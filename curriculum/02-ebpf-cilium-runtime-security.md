# Principal Platform Engineering Interview Curriculum

## Chapter 2 — eBPF, Cilium, Hubble, Falco, and Tetragon Runtime Security

> **Original scenario:** Design an eBPF-based runtime network-security architecture for a large Kubernetes platform. Explain where Cilium enforces policy, how Hubble provides evidence, how runtime detection differs from inline prevention, how you would use Falco and Tetragon, what fails under extreme load, and how you would migrate safely without creating a node-wide security or availability incident.

---

# 1. Why this question exists

This is not a test of whether you can say that eBPF is faster than iptables.

The interviewer is testing whether you understand:

- The Linux packet path before discussing Kubernetes networking.
- Where XDP, traffic control, cgroup, socket, tracepoint, kprobe, and LSM hooks execute.
- How eBPF programs are loaded, verified, attached, updated, and supplied with state.
- The difference between network enforcement, network observability, runtime detection, and runtime prevention.
- How Cilium maps Kubernetes workload labels into security identities.
- Why Cilium L7 policy can involve an Envoy userspace proxy rather than executing every decision inside eBPF.
- Why a compromised process using an already-allowed network path is not stopped by NetworkPolicy alone.
- Why asynchronous event collection can lose evidence under load.
- Why Tetragon `SIGKILL` and return-value override are not interchangeable.
- How kernel, agent, BPF-map, and policy failures can affect every workload on a node.
- How to canary a dataplane and enforcement migration safely.
- How to prove that a policy is both effective and operationally survivable.

A weak answer says:

> I would replace Calico with Cilium because eBPF is faster and use Falco for security.

A senior answer explains Cilium, Hubble, NetworkPolicy, and runtime alerts.

A Principal-level answer starts with the threat model and Linux execution path, separates each enforcement boundary, explains how evidence can be lost, uses multiple complementary controls, and bounds the impact of the security system itself.

---

# 2. The interview answer first

## 2.1 Ninety-second answer

> I would separate four concerns. Cilium provides the Kubernetes network dataplane, identity-aware L3/L4 policy, service load balancing, and selective L7 policy. Hubble provides flow evidence from the same dataplane. Falco provides broad behavioral detection from kernel event streams and plugins. Tetragon provides kernel-aware process, file, capability, and network tracing with optional inline enforcement.
>
> I would begin with the threat model and default-deny network segmentation. Cilium security identities are derived from workload labels, so policy remains stable as pod IPs change. I would use Kubernetes NetworkPolicy where portability is sufficient, CiliumNetworkPolicy for identity, DNS, host, or selected L7 requirements, and explicit egress controls for metadata services and external destinations. L7 policy is used only where its value justifies Envoy proxying and additional failure modes.
>
> Network policy is not runtime process control. If a compromised application is permitted to call the ledger service, Cilium cannot determine whether that allowed request was produced by legitimate code or a malicious shell. Falco can detect suspicious process behavior, but its kernel-to-userspace event buffers can overflow under load; dropped-event metrics are therefore a security SLI. For high-value prevention I would use a narrowly scoped Tetragon policy at an appropriate hook. Return-value override can prevent supported operations; `SIGKILL` terminates the process synchronously but does not guarantee that every triggering operation was prevented, so the hook and action must match the threat model.
>
> I would migrate through a canary node pool, validate kernel and BTF compatibility, run Cilium connectivity and policy-equivalence tests, observe Hubble flows, deploy runtime policies in monitor mode, and canary enforcement by node and workload. Stop conditions include packet loss, DNS regression, BPF-map pressure, verifier or program-load failure, unexpected policy denies, dropped runtime events, or customer-SLI degradation. The security platform must fail predictably and must never be allowed to disable or terminate an unbounded portion of the fleet.

## 2.2 Fifteen-second executive summary

> Cilium controls network reachability, Hubble explains network behavior, Falco detects suspicious behavior asynchronously, and Tetragon can enforce selected kernel operations inline. Use each for the boundary it actually controls, canary everything, and treat dropped evidence and enforcement blast radius as first-class risks.

---

# 3. Assumptions to state before designing

Use an opening such as:

> I will assume a multi-tenant Kubernetes platform with thousands of nodes, sensitive identity and payment services, untrusted application code, and a requirement to limit lateral movement without sacrificing availability. The cluster currently uses another CNI, so the design must include migration, rollback, policy equivalence, and kernel compatibility rather than only a greenfield target state.

Clarify or state assumptions about:

- Kubernetes distribution and version.
- Linux distribution and kernel versions.
- Managed versus self-managed nodes.
- Current CNI and kube-proxy mode.
- Overlay, native routing, or cloud VPC routing.
- Encryption requirements.
- Service mesh presence.
- Multi-cluster or ClusterMesh requirements.
- Tenant and regulatory boundaries.
- Which workloads may run privileged.
- Runtime-prevention requirements versus detection-only requirements.
- Security operations ownership and response time.
- Whether the organization can tolerate a fail-closed node dataplane.
- Required throughput, packets per second, and connection churn.

Do not assume that every environment should use the same policy mechanism. A payment-signing workload and a low-risk stateless catalog service can justify different enforcement depth.

---

# 4. Linux foundations from the ground up

## 4.1 User space and kernel space

Applications normally execute in user space. They request privileged operating-system work through system calls.

Examples:

```text
openat()     -> open a file
read()       -> read bytes
write()      -> write bytes
execve()     -> replace the current process image
connect()    -> initiate a socket connection
bind()       -> bind a local socket
setuid()     -> change user identity
bpf()        -> create/load/manage BPF objects
```

The kernel controls:

- Process scheduling.
- Virtual memory.
- Filesystems.
- Network stack.
- Device drivers.
- Credentials and capabilities.
- Namespaces and cgroups.
- Security hooks.

A Kubernetes pod does not bypass Linux. Containers are ordinary Linux processes constrained by namespaces, cgroups, capabilities, seccomp, and other security controls.

---

## 4.2 Simplified inbound packet path

```text
NIC
 |
 v
network driver
 |
 +--> XDP hook: very early packet processing before normal stack work
 |
 v
Linux networking stack creates/processes skb
 |
 +--> tc ingress hook
 |
 +--> routing / netfilter / conntrack / service translation
 |
 +--> cgroup and socket-related hooks where configured
 |
 v
socket receive queue
 |
 v
application read()/recv()
```

The exact path changes with:

- XDP mode.
- Native versus generic XDP.
- Overlay or direct routing.
- iptables/IPVS versus eBPF service load balancing.
- Host firewall.
- Socket-level acceleration.
- Local versus remote endpoint.
- Encryption.
- L7 proxy redirection.

The interview objective is not to recite every kernel function. It is to understand that different hooks see different context and impose different costs.

---

## 4.3 Simplified pod egress path

```text
application connect()/send()
 |
 v
socket / cgroup hooks
 |
 v
endpoint eBPF policy
 |
 +--> allow
 |      |
 |      +--> direct service/backend path
 |      +--> L7 proxy redirect when required
 |
 +--> deny and emit drop reason
 |
 v
routing / encapsulation / encryption
 |
 v
tc egress / host device
 |
 v
NIC
```

Questions to ask:

- Where is source identity attached or recovered?
- Which map contains policy state?
- Is service translation performed before or after policy?
- Does traffic pass through userspace Envoy?
- What happens when the Cilium agent is unavailable after programs are already loaded?
- What happens when policy regeneration fails?
- What evidence is emitted for a dropped packet?

---

# 5. What eBPF is

## 5.1 A programmable kernel execution environment

eBPF allows privileged software to load verified programs into the Linux kernel and attach them to supported hook points.

The program may:

- Inspect packet metadata.
- Permit, redirect, or drop traffic.
- Observe system calls and kernel functions.
- Read process credentials and namespace context.
- Update counters and state in maps.
- Emit selected events to user space.
- Enforce selected operations where a supported hook and action allow it.

The phrase “runs in the kernel” does not mean arbitrary unsafe kernel code is accepted. Programs are constrained by program type, helper availability, verifier rules, kernel features, and privileges.

---

## 5.2 Program lifecycle

```text
source code
  |
  v
Clang/LLVM compilation
  |
  v
eBPF bytecode + BTF/relocation metadata
  |
  v
privileged loader calls bpf() syscall
  |
  v
kernel verifier checks safety and validity
  |
  +--> reject with verifier log
  |
  v
program loaded, optionally JIT compiled
  |
  v
program attached to XDP/tc/cgroup/tracepoint/kprobe/LSM/etc.
  |
  v
program reads/writes maps and executes on matching events
```

A platform engineer must care about both compilation and runtime compatibility. A program that worked on one node image can fail to load on another due to kernel configuration, helper support, hook availability, BTF differences, or verifier behavior.

---

## 5.3 The verifier

The verifier analyzes a BPF program before the kernel accepts it.

Conceptually, it checks that:

- Memory access is valid.
- Pointers are used safely.
- Control flow is valid and bounded.
- Helper calls are allowed for the program type.
- Stack usage and state transitions meet constraints.
- The program will not execute arbitrary unsupported kernel operations.

A rejected program is an availability concern if the dataplane or security agent depends on it. Verifier failures must therefore be visible during AMI qualification and canary rollout.

Do not say:

> eBPF can never harm the kernel because the verifier makes it safe.

The verifier is a major safety boundary, but:

- Kernel and verifier vulnerabilities can exist.
- Privileged loaders remain security-critical.
- A logically valid program can still create severe performance or availability problems.
- A policy can be syntactically valid and operationally catastrophic.

---

## 5.4 Maps

BPF maps provide shared state between BPF programs and user-space control agents.

Typical uses include:

- Endpoint identity.
- Policy decisions.
- Service and backend tables.
- Connection tracking.
- NAT state.
- Metrics and counters.
- Program tail-call tables.
- Runtime-policy selectors.

Map types have different properties: hash, array, LRU, per-CPU, ring buffer, perf event array, and others.

Maps matter operationally because they consume kernel memory and have finite capacity. Symptoms of map pressure include:

- Failed inserts.
- Connection failures.
- NAT or conntrack errors.
- Policy inconsistency.
- Increased contention.
- Agent warnings.

Cilium exposes map-operation and map-pressure metrics. A production design treats map sizing as capacity engineering rather than a hidden default.

---

## 5.5 Helpers

BPF programs cannot call arbitrary kernel functions. They use helper functions or other supported mechanisms appropriate to the program type.

Helpers may allow a program to:

- Read or update a map.
- Obtain time.
- Redirect a packet.
- Adjust packet data.
- Retrieve current process or cgroup information.
- Emit an event.
- Access socket metadata.

Available helpers depend on the kernel, program type, and security policy.

---

## 5.6 Tail calls

Tail calls allow one BPF program to transfer execution to another program through a program-array map.

This helps decompose a large datapath into stages, but introduces operational questions:

- Did the target program load successfully?
- Is the program-array entry populated?
- Is the tail-call limit approached?
- Does an upgrade leave incompatible program stages?

---

## 5.7 JIT compilation

The kernel may JIT-compile eBPF bytecode into native machine instructions.

Benefits can include lower execution overhead, but security and operations must consider:

- JIT hardening settings.
- Kernel configuration.
- Architecture differences.
- Debugging visibility.
- The performance cost of hardening options.

---

## 5.8 BTF and CO-RE

BPF Type Format provides type information used by tooling, loaders, debugging, and relocation.

Compile Once – Run Everywhere, through libbpf CO-RE, combines compiled BPF objects, BTF information, and relocations so one object can adapt to multiple compatible kernel layouts.

CO-RE improves portability; it does not mean every kernel supports every hook, helper, configuration option, or enforcement action. Maintain a tested kernel compatibility matrix.

---

# 6. Important eBPF hook points

## 6.1 XDP

XDP executes very early in the receive path, close to the network driver and before the normal socket-buffer path.

Good uses:

- Early packet drops.
- DDoS filtering.
- Fast redirection.
- Simple high-rate classification.

Trade-offs:

- Less protocol and process context than later hooks.
- Driver-mode compatibility varies.
- Incorrect policy can drop traffic before richer observability exists.
- Generic XDP and native XDP have different behavior and performance.

Principal explanation:

> XDP is powerful when the decision can be made from early packet context. It is not automatically the best hook for identity-rich application policy.

---

## 6.2 Traffic control ingress and egress

The tc hooks execute later in the network path on an interface and have access to an `skb` representation.

Cilium uses tc hooks extensively for:

- Endpoint policy.
- Routing and forwarding.
- Service behavior.
- Encapsulation.
- Identity-aware decisions.
- Packet metadata handling.

Compared with XDP, tc generally has richer stack context but executes after more kernel processing.

---

## 6.3 Cgroup and socket hooks

Cgroup and socket hooks can associate policy with process groups and sockets.

They can support:

- Socket-level policy.
- Connection-time decisions.
- Local socket acceleration.
- Efficient service redirection.
- Workload/cgroup context.

These hooks are important because a network decision can sometimes be made before every packet traverses the full datapath.

---

## 6.4 Tracepoints

Tracepoints are kernel-defined instrumentation points with relatively stable semantics.

They are useful for observation of:

- System calls.
- Scheduling.
- Network events.
- Process lifecycle.
- Other kernel subsystems.

A runtime detector can use tracepoints to collect events, but those events usually still need transfer to user space for rule evaluation unless filtering or action occurs in the kernel.

---

## 6.5 Kprobes

Kprobes dynamically attach to kernel functions.

Advantages:

- Broad observability.
- Ability to inspect specific internal paths.

Risks:

- Kernel function names and semantics can change.
- Arguments require exact understanding.
- Policies can be vulnerable to time-of-check/time-of-use mistakes.
- Enforcement support is not universal.

---

## 6.6 LSM hooks

Linux Security Module hooks are purpose-built security decision points.

They are often more appropriate for prevention because the kernel is explicitly asking whether an operation should be allowed.

Examples of security domains include:

- File access.
- Process execution.
- Credentials.
- Network operations.

LSM-based policy should complement, not casually replace:

- seccomp.
- AppArmor.
- SELinux.
- Linux capabilities.
- read-only filesystems.
- Kubernetes Pod Security controls.

---

# 7. Cilium architecture

## 7.1 Main components

```text
Kubernetes API
   |
   +--> Cilium operator
   |
   +--> Cilium agent DaemonSet on every node
             |
             +--> endpoint discovery and identity
             +--> policy calculation and regeneration
             +--> BPF program loading/attachment
             +--> BPF map management
             +--> optional Envoy L7 proxy integration
             +--> embedded Hubble server

Hubble Relay
   |
   +--> aggregates per-node Hubble streams
   |
   +--> CLI, UI, metrics, and consumers
```

Depending on the deployment, additional components may include:

- Envoy DaemonSet or embedded proxy mode.
- ClusterMesh components.
- Cilium Gateway API or ingress components.
- Encryption configuration.
- Tetragon as a separate DaemonSet.

---

## 7.2 Security identities

Cilium derives security identities from relevant labels rather than treating the pod IP as the durable identity.

Conceptually:

```text
Pod labels / namespace labels / selected identity labels
        |
        v
security identity allocation
        |
        v
numeric identity used by datapath and policy maps
```

This means policy can express:

> `app=checkout` in namespace `store` may call `app=ledger` in namespace `payments` on TCP 8443.

The policy remains stable as pods restart and receive different IP addresses.

Important nuances:

- Not every label should necessarily participate in identity.
- High-cardinality identity labels can increase identity and policy scale.
- Identity propagation and endpoint regeneration have convergence time.
- An identity of zero in Hubble can indicate that identity was not found and should be investigated.
- Multi-cluster identity allocation has additional design limits and ownership requirements.

---

## 7.3 Endpoint policy

Cilium endpoint policy uses identity and policy state to decide whether traffic is allowed.

At a high level:

```text
packet arrives for endpoint
       |
       v
recover source/destination identity
       |
       v
lookup endpoint policy map
       |
       +--> allow L3/L4 path
       +--> redirect to L7 proxy
       +--> deny and emit verdict/drop reason
```

A NetworkPolicy decision answers whether a communication path is permitted. It does not prove that the process creating the connection is behaving legitimately.

---

## 7.4 Service load balancing

Cilium can implement Kubernetes service load balancing through BPF maps and programs, including kube-proxy replacement modes where configured.

This introduces benefits and risks:

Benefits:

- Reduced dependence on large iptables rule sets.
- Programmable service handling.
- Socket-level acceleration in supported paths.
- Unified visibility and service translation.

Risks:

- Service-map and backend-map capacity.
- Upgrade compatibility.
- Node-wide impact of datapath defects.
- Operational dependence on Cilium agent and programs.
- Different behavior from kube-proxy that must be tested.

Do not claim that kube-proxy replacement is mandatory to benefit from Cilium. Choose it as a separate architectural decision.

---

## 7.5 L7 policy and Envoy

Cilium can redirect selected traffic to Envoy for application-layer policy.

Examples:

- HTTP method and path.
- Kafka-aware rules.
- DNS policy support through proxy mechanisms.
- Gateway and service-mesh functions.

Important correction:

> Not every Cilium L7 decision runs entirely inside eBPF.

The eBPF datapath can redirect matching traffic to userspace Envoy, where protocol parsing and L7 enforcement occur.

Therefore L7 policy adds:

- A userspace proxy data path.
- Proxy CPU and memory.
- Connection and queue limits.
- Certificate and protocol concerns.
- Additional observability.
- Additional failure modes.

Use L7 policy selectively where it provides material risk reduction.

---

# 8. Network-policy design

## 8.1 Start with the communication contract

For each service, define:

- Owner.
- Business criticality.
- Inbound callers.
- Outbound dependencies.
- Ports and protocols.
- External domains.
- Administrative paths.
- Disaster-recovery paths.
- Batch and cron behavior.
- Data classification.

Do not generate permanent allow rules automatically from every observed flow. Observability can include accidental, compromised, or one-time traffic.

Use observed traffic as evidence, then require owner approval.

---

## 8.2 Policy layers

### Kubernetes NetworkPolicy

Use when:

- Standard ingress/egress L3/L4 policy is enough.
- Portability is important.
- The desired semantics are supported consistently by the chosen CNI.

### CiliumNetworkPolicy

Use when you require Cilium-specific controls such as:

- Extended identity selectors.
- Entities.
- FQDN-aware egress.
- Selected L7 rules.
- ICMP controls.
- Node or host interactions.
- Additional Cilium policy semantics.

### CiliumClusterwideNetworkPolicy

Use for reviewed cluster-wide rules such as:

- Platform DNS.
- Monitoring and security agents.
- Selected host controls.
- Shared infrastructure services.

Cluster-wide policy has large blast radius. Require ownership, testing, and progressive rollout.

---

## 8.3 Default deny

Default deny means a selected endpoint receives only explicitly permitted traffic.

It is not enough to say “enable default deny.” You must define:

- Which endpoints are selected.
- Ingress versus egress.
- DNS access.
- Control-plane access.
- Health checks.
- Metrics and logging.
- Service mesh and sidecar paths.
- Node-local services.
- Emergency and maintenance paths.

A default-deny rollout without a dependency inventory can become a broad outage.

---

## 8.4 Example identity-aware policy

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: checkout-api
  namespace: store
spec:
  endpointSelector:
    matchLabels:
      app: checkout-api

  ingress:
    - fromEndpoints:
        - matchLabels:
            k8s:io.kubernetes.pod.namespace: edge
            app: edge-gateway
      toPorts:
        - ports:
            - port: "8443"
              protocol: TCP

  egress:
    - toEndpoints:
        - matchLabels:
            k8s:io.kubernetes.pod.namespace: identity
            app: identity-api
      toPorts:
        - ports:
            - port: "8443"
              protocol: TCP

    - toEndpoints:
        - matchLabels:
            k8s:io.kubernetes.pod.namespace: payments
            app: ledger
      toPorts:
        - ports:
            - port: "8443"
              protocol: TCP
```

This policy expresses permitted network paths. It does not prove the request is authorized at the application layer. Authentication and authorization still belong in workload identity, mTLS, tokens, and application policy.

---

## 8.5 Selected L7 policy

Conceptual example:

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: playback-l7
  namespace: playback
spec:
  endpointSelector:
    matchLabels:
      app: playback-api
  ingress:
    - fromEndpoints:
        - matchLabels:
            k8s:io.kubernetes.pod.namespace: edge
            app: edge-gateway
      toPorts:
        - ports:
            - port: "8080"
              protocol: TCP
          rules:
            http:
              - method: GET
                path: "^/v1/(manifest|segment)/.*"
```

Questions before using it:

- Is the traffic plaintext at the proxy point or can Cilium decrypt it safely?
- Does this add Envoy to the data path?
- What happens if Envoy is unavailable or saturated?
- Are protocol upgrades, streaming, gRPC, or WebSocket behavior supported?
- Is application authorization already a stronger control?
- What is the measurable risk reduction?

---

# 9. Hubble: network evidence

## 9.1 What Hubble provides

Hubble builds on Cilium datapath events to provide visibility into:

- Source and destination identity.
- Forwarded and dropped flows.
- Policy verdicts.
- DNS queries and responses.
- TCP behavior.
- Selected L7 protocol information.
- Service dependencies.
- Latency and error patterns.

Hubble server is embedded in the Cilium agent. Hubble Relay connects to node-local Hubble servers to provide cluster-wide visibility.

---

## 9.2 What Hubble can prove

Hubble can help answer:

- Was the packet dropped by policy?
- Which source identity attempted the connection?
- Which destination and port were involved?
- Was the failure at DNS, TCP, or selected L7 processing?
- Did the request reach the expected service?
- Are connections timing out or being reset?
- Which policy verdict changed after rollout?

---

## 9.3 What Hubble cannot prove by itself

- That a process is legitimate.
- That an allowed request is authorized by business logic.
- That no event was lost.
- That a user journey completed correctly.
- That encrypted payload content was safe.
- That an attacker did not use an already-permitted path.
- That the kernel itself is uncompromised.

Hubble is network evidence, not a universal runtime-security system.

---

# 10. Runtime security layers

## 10.1 Preventive Linux controls

Before adding an eBPF runtime tool, use foundational controls:

- Non-root containers.
- Minimal Linux capabilities.
- Read-only root filesystem where possible.
- Seccomp profiles.
- AppArmor or SELinux.
- User namespaces where supported.
- Pod Security Admission.
- Signed and scanned images.
- Immutable or minimal node images.
- No unnecessary hostPath, hostNetwork, hostPID, or privileged pods.

These controls reduce what a compromised process can do without depending on event detection.

---

## 10.2 Network enforcement

Cilium limits network reachability.

It can reduce:

- Lateral movement.
- Unapproved egress.
- Metadata-service access.
- Cross-namespace exposure.
- Unexpected inbound paths.

But an allowed network connection can still carry malicious behavior.

Example:

```text
checkout-api is allowed to call ledger:8443
```

If checkout-api is compromised, NetworkPolicy alone cannot distinguish a legitimate payment call from an attacker using the same socket path. Application authorization and runtime controls are still required.

---

## 10.3 Behavioral detection with Falco

Falco evaluates events from supported event sources against rules and emits alerts.

Typical detections include:

- Shell spawned in a container.
- Unexpected package-management command.
- Sensitive file access.
- Privilege escalation behavior.
- Container escape indicators.
- Unexpected outbound tools.
- Kubernetes audit activity through plugins.

Falco is valuable because its mature rule ecosystem and event model cover many behaviors.

However, syscall and kernel-event processing generally includes a kernel-to-userspace event path. Under load, buffers can fill and events can be dropped. Falco explicitly exposes dropped-event behavior and configurable actions.

Principal conclusion:

> A detection system that can drop evidence must expose evidence-loss metrics, and its alerting SLO must include both detection latency and event completeness.

---

## 10.4 Tetragon observation and enforcement

Tetragon uses eBPF for process, file, network, capability, and kernel-event visibility. TracingPolicies can select hook points, filter in the kernel, emit events, and optionally take actions.

Potential advantages:

- Kubernetes-aware process context.
- In-kernel filtering reduces unnecessary event export.
- Selected enforcement can happen inline with the operation.
- Policies can be scoped by workload and node selectors.
- Persistent enforcement can keep selected sensors attached during userspace-agent interruption when explicitly configured, though event reporting is unavailable during the interruption.

Risks:

- TracingPolicy is low-level and powerful.
- Hook semantics vary.
- Incorrect arguments or selectors can create false confidence.
- Enforcement can terminate critical workloads.
- Kernel compatibility matters.
- Time-of-check/time-of-use issues can exist.
- A policy that works on one kernel may behave differently on another.

---

# 11. Detection versus inline prevention

## 11.1 Asynchronous detection path

```text
kernel event
   |
   v
kernel/eBPF capture
   |
   v
shared buffer
   |
   v
userspace reader
   |
   v
rule evaluation / enrichment
   |
   v
alert output
   |
   v
human or automation response
```

Failure points:

- Buffer full.
- Reader CPU starvation.
- Event parser backlog.
- Rule-engine saturation.
- Output queue failure.
- SIEM ingestion delay.
- Alert throttling.
- Response automation failure.

Detection is still valuable, but it must not be described as guaranteed prevention.

---

## 11.2 Inline enforcement path

```text
process attempts operation
        |
        v
selected kernel hook executes policy
        |
        +--> allow
        |
        +--> return error / override supported operation
        |
        +--> send signal / terminate process
        |
        +--> other supported action
```

The security decision occurs as part of the kernel operation path rather than after a user-space alert round trip.

This reduces response latency, but it increases the consequence of policy mistakes.

---

## 11.3 Tetragon override versus signal

### Return-value override

A supported override changes the function return path so the original operation does not proceed as it normally would.

Use when:

- The selected hook/function supports override.
- Preventing the operation is required.
- Kernel configuration supports the mechanism.
- The application can safely handle the returned error.

Limitations:

- Only supported functions and hooks can be overridden.
- Kernel configuration matters.
- An incorrect return value can produce unexpected application behavior.
- You must understand exactly where the hook sits in the operation.

### `SIGKILL` or signal action

A signal action terminates the matching process synchronously.

Important correction:

> Killing the process does not guarantee that every triggering operation was prevented.

For example, if a signal is sent during a write path, data may already have been written. Where operation prevention is required, use an appropriate override or security hook and combine actions only when justified.

Principal statement:

> I choose the enforcement hook and action from the exact invariant I need to preserve. “Kill the process” and “prevent the operation” are different guarantees.

---

# 12. Threat model

A strong design maps each threat to the control that can actually reduce it.

| Threat | Primary controls | Evidence |
|---|---|---|
| Internet attacker reaching an internal service | ingress, gateway auth, Cilium policy, application auth | Hubble, gateway logs, traces |
| Compromised pod scanning lateral services | default-deny Cilium policy | Hubble denied flows |
| Compromised pod calling an already-allowed API maliciously | application authorization, least privilege, runtime detection/prevention | traces, audit logs, Tetragon/Falco |
| Shell spawned in production container | no shell in image, seccomp/MAC, Falco/Tetragon detection, selected enforcement | process-exec events |
| Sensitive file read | read-only filesystem, AppArmor/SELinux, secrets design, selected runtime policy | file events, audit trail |
| Privilege escalation | drop capabilities, seccomp, no privileged pods, admission policy, runtime detection | capability and process events |
| Metadata credential theft | Cilium egress/host policy, cloud workload identity, IMDS protection | Hubble and cloud audit |
| Data exfiltration | egress policy, proxy, DLP/application control, runtime detection | Hubble, DNS, proxy logs |
| Kernel exploit | patching, minimal kernel attack surface, node isolation, detection | node telemetry, security response |
| Malicious policy rollout | Git review, policy testing, canary, bounded enforcement | Git history, policy metrics |

No single tool covers every row.

---

# 13. Reference architecture

```text
                         POLICY AND SUPPLY CHAIN

Git -> review -> CI policy tests -> signed artifacts -> progressive deployment

                         KUBERNETES CONTROL PLANE

NetworkPolicy / CiliumNetworkPolicy / TracingPolicy / admission controls
                 |                         |
                 v                         v

+---------------------------------------------------------------------+
| Node                                                                |
|                                                                     |
|  Cilium agent                  Tetragon agent         Falco agent    |
|      |                              |                    |           |
|      | loads/manages                | loads/manages      | reads     |
|      v                              v                    v           |
|  Cilium BPF programs          runtime BPF programs   event buffers   |
|      |                              |                    |           |
|      +--> L3/L4 policy              +--> observe         +--> alerts |
|      +--> service LB                +--> filter                      |
|      +--> routing                   +--> selected enforce            |
|      +--> optional Envoy L7                                       |
|                                                                     |
|  Pods / processes / sockets / files / network interfaces            |
+---------------------------------------------------------------------+
                 |                         |
                 v                         v
            Hubble server             Tetragon events
                 |
                 v
            Hubble Relay -> dashboards, SIEM, incident tooling
```

Design principles:

- Network controls remain independent of runtime detection output.
- Runtime enforcement is narrower than runtime observation.
- Policies are generated and reviewed as code.
- Agents and programs expose health, version, and loss metrics.
- A policy rollback path exists even during an incident.
- Enforcement cannot target the whole fleet without explicit safeguards.

---

# 14. Runtime-policy design

## 14.1 Begin in observation mode

Before enforcement:

1. Observe the workload over a representative cycle.
2. Include deployments, backups, failover, maintenance, and incident tooling.
3. Classify expected process trees.
4. Identify interpreters, package managers, shells, and administrative commands.
5. Verify image digests and binary paths.
6. Measure event rate and dropped events.
7. Build a response runbook.

Observation must include rare but legitimate behavior. A policy trained only during normal weekday traffic can terminate disaster-recovery or certificate-rotation processes.

---

## 14.2 Prefer high-confidence invariants

Good candidates for narrow prevention:

- A production workload must never load a kernel module.
- A restricted container must never invoke the `bpf()` syscall.
- A payment-signing workload must never execute an interactive shell.
- An application container must not write to a protected host path.
- A workload must not open a raw packet socket.
- A namespace must not change process credentials beyond an allowed set.

Poor initial prevention candidates:

- Any unusual command line.
- Any process not seen during a short observation window.
- Broad file-path patterns with known bypasses.
- Low-level kernel functions whose semantics are not understood.
- Policies applied cluster-wide before canary validation.

---

## 14.3 Conceptual Tetragon enforcement pattern

The exact hook and argument schema must be verified against the deployed Tetragon and kernel versions.

```yaml
apiVersion: cilium.io/v1alpha1
kind: TracingPolicy
metadata:
  name: restricted-operation
spec:
  # Choose a supported kprobe or LSM hook that represents the invariant.
  # Scope with workload/node selectors.
  # Begin in monitor mode where supported.
  # Apply an Override action when the operation itself must be prevented.
  # Add Sigkill only when terminating the actor is also required.
```

Do not copy a low-level enforcement example into production simply because it works in a demo. Prove:

- The hook covers all relevant operation paths.
- Alternate syscalls or APIs do not bypass it.
- The action occurs before the protected side effect when required.
- The policy is scoped to the intended workloads.
- The kernel supports the action.
- The application and platform remain stable when the operation is denied.

---

# 15. Failure modes and catastrophic stress

## 15.1 BPF program-load failure

Causes:

- Kernel feature missing.
- Verifier rejection.
- BTF mismatch.
- Unsupported helper.
- Insufficient privileges.
- Object or map incompatibility.
- Program too complex for the verifier.
- Node image drift.

Impact depends on the component:

- Cilium agent may fail to initialize the endpoint datapath.
- Network policy may not be regenerated.
- Node may remain NotReady or workloads may lose connectivity.
- Runtime enforcement may be absent.

Required behavior:

- Fail visibly.
- Taint or exclude incompatible nodes.
- Stop rollout automatically.
- Preserve verifier logs.
- Roll back to known-good agent or AMI.

---

## 15.2 BPF-map pressure

Examples:

- Conntrack table exhaustion.
- NAT map exhaustion.
- Service/backend map limits.
- Policy-map pressure.
- Identity growth.

Symptoms:

- New connections fail while established ones survive.
- NAT errors.
- Intermittent service reachability.
- Policy-map insertion warnings.
- Increased CPU or lock contention.

Monitor:

- Cilium BPF-map pressure.
- Map operations and failures.
- Connection and NAT metrics.
- Workload connection churn.
- Kernel memory.

Do not only increase map sizes. Determine whether the cause is legitimate scale, retry storms, scans, leaks, or an attack.

---

## 15.3 Cilium-agent failure

Already-loaded BPF programs may continue forwarding according to existing state, but the node loses control-plane functions such as:

- Endpoint updates.
- Policy regeneration.
- Service/backend updates.
- Identity synchronization.
- New pod setup.
- Observability processing.

This resembles other control-plane/data-plane separations: stale dataplane behavior can continue temporarily, but freshness and new workload lifecycle degrade.

Define:

- Maximum tolerated agent outage.
- How new pods are prevented from landing on an unhealthy node.
- Whether existing endpoints continue safely.
- When the node is cordoned or replaced.

---

## 15.4 Bad cluster-wide policy

A malformed or overbroad cluster-wide policy can:

- Block DNS.
- Block kube-apiserver access.
- Block node health.
- Block monitoring and security agents.
- Interrupt every tenant.

Controls:

- Static policy validation.
- Unit and integration tests.
- Policy audit mode where supported.
- Canary nodes or namespaces.
- Explicit platform-service allow rules.
- Progressive rollout.
- Emergency rollback.
- Admission restrictions on who can create cluster-wide policy.

---

## 15.5 L7 proxy saturation

Selected Cilium L7 policy can introduce Envoy saturation:

- CPU throttling.
- Memory pressure.
- Connection-pool exhaustion.
- Pending-request overflow.
- Protocol parsing errors.
- Certificate problems.
- Tail-latency increase.

The symptom may look like a network-policy problem even though L3/L4 forwarding is healthy.

Debug both:

- Cilium policy verdict.
- Envoy listener, cluster, route, and proxy metrics.

---

## 15.6 Hubble observability loss

Possible causes:

- Ring-buffer or event-buffer pressure.
- Agent overload.
- Relay disconnection.
- Exporter backpressure.
- Metric-cardinality explosion.
- Sampling or retention limits.

A missing Hubble event does not prove that traffic did not occur. Monitor observability pipeline health and distinguish “zero events” from “no evidence available.”

---

## 15.7 Falco dropped events

Under high syscall rates or slow userspace processing, kernel-to-userspace buffers may fill and events may be dropped.

Examples of load:

- Massive CI/CD builds.
- Package installation.
- Container startup storms.
- DDoS-induced process/network activity.
- High file-I/O workloads.
- Broad, expensive rule sets.

Controls:

- Monitor dropped-event metrics and alerts.
- Tune buffers carefully.
- Reduce unnecessary syscall collection.
- Optimize rules.
- Separate high-volume build pools from sensitive production pools.
- Do not call detection complete when event loss is nonzero.

---

## 15.8 Tetragon enforcement mistake

A bad policy can:

- Kill every process matching a common syscall.
- Break kubelet, containerd, Cilium, DNS, or monitoring.
- Trigger restart loops.
- Prevent recovery tooling from running.
- Apply differently across kernel versions.

Required safeguards:

- Node selector.
- Pod/namespace selector.
- Workload criticality exclusion.
- Canary node pool.
- Monitor mode first.
- Maximum affected workload threshold.
- Emergency policy-removal path.
- Out-of-band node access or replacement.
- Policy owner and expiry.

---

## 15.9 Kernel or node regression

A dataplane upgrade may expose:

- Kernel panic.
- Soft lockup.
- Network-driver issue.
- BPF JIT regression.
- BTF incompatibility.
- Map contention under stress.
- Increased packet latency.

Canary with realistic packets-per-second, connection churn, encryption, and failure injection. Functional connectivity tests alone are insufficient.

---

# 16. Observability and security SLIs

## 16.1 Cilium health

Monitor:

- Agent readiness and restart rate.
- Endpoint regeneration failures.
- Policy-revision convergence.
- BPF program-load errors.
- BPF map operations and pressure.
- Conntrack/NAT errors.
- Packet drops by reason.
- Service/backend counts.
- Identity allocation and synchronization.
- Envoy/L7 proxy health where used.
- Cilium operator health.

---

## 16.2 Hubble health

Monitor:

- Flow event rate.
- Dropped-flow rate.
- Relay connectivity.
- Export queue health.
- DNS and L7 visibility availability.
- Per-node coverage.
- Data freshness.
- Storage and SIEM ingestion latency.

---

## 16.3 Falco health

Monitor:

- Kernel driver/eBPF probe health.
- Event rate.
- Dropped syscall events.
- Rule evaluation latency.
- Output queue failures.
- Alert delivery latency.
- Plugin health.
- Resource saturation.

---

## 16.4 Tetragon health

Monitor:

- Agent readiness.
- Sensor/policy load success.
- Policy version by node.
- Event rate and export health.
- Enforcement action count.
- Unexpected process terminations.
- Kernel/program-load errors.
- Policy scope and selector match counts.
- Persistent-enforcement status when used.

---

## 16.5 Product and platform SLIs

Security telemetry must be correlated with:

- Request success.
- DNS latency.
- TCP connection success.
- Packet loss and retransmission.
- Pod startup time.
- Node readiness.
- Deployment success.
- P99 latency.
- Customer journey completion.

A policy that blocks an attack but causes a broad production outage has failed the availability requirement. A policy that preserves availability but silently permits prohibited behavior has failed the security requirement.

---

# 17. Debugging methodology

## 17.1 Overall Cilium status

```bash
cilium status --verbose
```

Use it to identify:

- Unhealthy agents.
- Operator health.
- Hubble status.
- Version skew.
- Cluster-wide warnings.

Then run:

```bash
cilium connectivity test
```

This tests representative paths, but it does not prove every application dependency or production traffic pattern.

---

## 17.2 Observe denied flows

```bash
hubble observe --verdict DROPPED --follow
```

Scope the query:

```bash
hubble observe \
  --from-namespace store \
  --from-pod checkout-api \
  --verdict DROPPED \
  --follow
```

Questions answered:

- Which identity attempted traffic?
- Which destination and port?
- Was the verdict policy-related?
- Did the failure occur before an application response?

---

## 17.3 Follow one workload

```bash
hubble observe \
  --from-pod store/checkout-api \
  --follow
```

Correlate with:

- Application request ID.
- Trace ID.
- Destination service.
- DNS event.
- TCP flags.
- Policy verdict.

---

## 17.4 Inspect endpoints and effective policy

Run inside the Cilium agent pod or using the supported debug interface for the deployed version:

```bash
cilium-dbg endpoint list
cilium-dbg endpoint get <endpoint-id>
cilium-dbg policy get
```

Use endpoint labels from `endpoint get` to identify the source policy affecting the workload.

Questions:

- Is the endpoint ready?
- Which security identity is assigned?
- Which ingress and egress policy revision is active?
- Did policy regeneration fail?
- Is audit mode enabled?

---

## 17.5 Inspect drops

```bash
cilium-dbg monitor --type drop
```

Use this for low-level node-local drop reasons.

Do not run highly verbose monitoring indefinitely on a stressed production node. Scope the observation and understand its overhead.

---

## 17.6 Inspect BPF objects

```bash
bpftool prog show
bpftool map show
bpftool net
```

Useful questions:

- Are expected programs loaded and attached?
- Are map sizes and entry counts plausible?
- Is a program missing on one node?
- Did a rollout leave mixed versions?

Cilium-specific commands and pinned paths vary by version. Use the exact release documentation and support tooling.

---

## 17.7 Inspect Cilium logs

```bash
kubectl -n kube-system logs ds/cilium -c cilium-agent --since=20m
kubectl -n kube-system get pods -l k8s-app=cilium -o wide
```

Look for:

- Verifier errors.
- Program-load failure.
- Endpoint regeneration failure.
- Map allocation/insertion errors.
- Identity synchronization problems.
- API connectivity failure.
- Proxy errors.

---

## 17.8 Hubble relay and coverage

```bash
kubectl -n kube-system get pods -l k8s-app=hubble-relay
kubectl -n kube-system logs deploy/hubble-relay --since=20m
```

Verify that missing flows are not caused by relay or export failure.

---

## 17.9 Tetragon events

```bash
tetra getevents -o compact
```

Scope by namespace, pod, process, or event type where supported by the deployed CLI.

Correlate:

- Parent process.
- Binary.
- Arguments.
- Container and pod identity.
- Node.
- Policy action.
- Exit signal or return result.

---

## 17.10 Falco dropped events

Check Falco metrics and logs for syscall-event drops.

A log such as “buffer was full” changes the incident conclusion:

> We have evidence of suspicious activity, but we do not have complete event coverage for the affected interval.

Do not represent incomplete telemetry as a clean bill of health.

---

# 18. Incident scenario

## Scenario

During a large CI/CD release, security alerts report shells and unexpected outbound connections from production pods. At the same time, Falco reports dropped syscall events, Hubble shows a spike in denied egress, and several nodes experience increased Cilium map pressure.

## STABILIZE response

### Stop the blast radius

- Pause the release and image promotion.
- Freeze network and runtime-policy changes.
- Isolate affected namespaces or node pools.
- Revoke compromised workload credentials if evidence supports compromise.
- Preserve node, container, image, and audit evidence.
- Assign incident command and security lead.

Do not immediately reboot every node. That can erase evidence and create a larger connection and scheduling storm.

---

### Trace one representative event

Choose one affected pod and build a timeline:

```text
image deployment
 -> process start
 -> unexpected shell or binary
 -> DNS query
 -> connection attempt
 -> Cilium verdict
 -> Tetragon/Falco event
 -> credential or API activity
```

Ask:

- Was the process part of the image or introduced at runtime?
- Was the network attempt allowed or denied?
- Was the target already permitted by policy?
- Did runtime telemetry show the parent process?
- Were events dropped during the interval?
- Did an enforcement action occur?

---

### Bound the failure

Group by:

- Image digest.
- Namespace and service account.
- Node pool and AMI.
- Deployment revision.
- Cluster and region.
- Binary path and hash.
- Destination domain or IP.
- Cilium identity.
- Runtime policy version.

This distinguishes one compromised workload from a shared supply-chain or node-level event.

---

### Inspect authoritative evidence

```bash
cilium status --verbose
hubble observe --from-pod <ns>/<pod> --follow
hubble observe --verdict DROPPED --since 30m
kubectl get cnp,ccnp,networkpolicy -A
kubectl get pod <pod> -n <ns> -o yaml
kubectl get events -A --sort-by=.lastTimestamp
```

On affected nodes:

```bash
bpftool prog show
bpftool map show
journalctl -u kubelet --since '-30 min'
journalctl -u containerd --since '-30 min'
```

Runtime evidence:

```bash
tetra getevents -o compact
```

Review Falco event-drop counters and output-pipeline health.

Cloud evidence:

- Workload identity and token use.
- API audit logs.
- Registry and image attestations.
- Network flow logs.
- Secret access.

---

### Competing hypotheses

1. Legitimate release tooling spawned shells and generated high event volume.
2. Compromised image executed an unauthorized payload.
3. A compromised pod used an allowed dependency path.
4. Runtime-policy rollout generated false positives.
5. Cilium policy or identity convergence caused unexpected denies.
6. Retry or connection storm caused BPF-map pressure independently of the security event.
7. A node-level compromise affected multiple pods.

Use evidence to eliminate hypotheses rather than choosing the most dramatic explanation first.

---

### Limit harm

Possible mitigations:

- Roll back the image revision.
- Quarantine affected workloads through a narrow policy.
- Remove service-account permissions.
- Block confirmed malicious destinations.
- Shift traffic to known-good pools.
- Reduce release concurrency causing map and event pressure.
- Apply a narrowly scoped, validated Tetragon enforcement policy to the confirmed invariant.

Do not deploy a broad “kill every shell” rule cluster-wide during the incident without proving that platform recovery and administrative tooling will remain functional.

---

### Root cause and contributing conditions

Example:

- Trigger: compromised build artifact introduced an unauthorized binary.
- Root systemic weakness: image-attestation policy was not enforced.
- Containment weakness: workload had broad egress and excessive API permissions.
- Detection weakness: Falco event loss occurred during the highest-rate interval.
- Capacity weakness: retry storm increased Cilium conntrack/map pressure.
- Repair risk: a cluster-wide Tetragon kill policy would have terminated legitimate support tooling.

---

### Permanent correction

- Enforce signed image and provenance policy.
- Reduce workload permissions.
- Narrow Cilium egress dependencies.
- Separate high-volume build nodes from production nodes.
- Add event-loss SLOs for runtime sensors.
- Add map-pressure capacity tests.
- Add high-confidence Tetragon prevention for the protected invariant.
- Test policy rollback and node replacement.
- Conduct a forensic and detection-coverage review.

---

# 19. Migration strategy

## Phase 1 — Threat model and baseline

Collect:

- Existing network-policy coverage.
- Service dependency graph.
- Packets per second and connection churn.
- DNS volume.
- NAT and conntrack size.
- Current latency, loss, and retransmission.
- Kernel and AMI matrix.
- Existing runtime event rate.
- Current security incidents and gaps.

---

## Phase 2 — Build a compatibility matrix

Test each supported node image for:

- Required BPF features.
- BTF availability.
- XDP/tc/cgroup hooks.
- Cgroup version.
- Encryption support.
- Kernel configuration for selected Tetragon actions.
- Verifier behavior.
- Cilium and Tetragon supported versions.
- Cloud NIC and MTU behavior.

Commands may include:

```bash
uname -a
bpftool feature probe kernel
bpftool prog show
bpftool map show
```

Record results as an AMI certification artifact.

---

## Phase 3 — Canary Cilium dataplane

Use a dedicated node pool:

- Explicit taints and labels.
- Representative test workloads.
- Small production traffic percentage.
- Separate rollback capacity.
- Same observability as the existing dataplane.

Validate:

- Pod-to-pod.
- Service access.
- DNS.
- NetworkPolicy.
- NodePort/load balancer.
- HostNetwork and node-local services.
- CSI and storage.
- Service mesh.
- Encryption.
- Drain and reboot.
- Node replacement.

---

## Phase 4 — Policy equivalence

For every existing policy:

- Compare intended semantics.
- Test ingress and egress.
- Test established connections.
- Test DNS and external destinations.
- Test rare operational paths.
- Compare denied-flow evidence.

Do not assume two CNI implementations interpret every edge case identically.

---

## Phase 5 — Hubble observation

Use Hubble to identify:

- Required dependencies.
- Unexpected cross-namespace traffic.
- DNS behavior.
- Denied flows.
- L7 value candidates.
- High connection churn.

Require service owners to approve dependency contracts.

---

## Phase 6 — Default-deny canaries

Progress through:

1. Low-risk stateless namespace.
2. Internal service.
3. Customer-facing service with rollback.
4. High-value service after full synthetic validation.

Stop on:

- Customer-SLI regression.
- Unexplained denies.
- DNS failure.
- Endpoint regeneration errors.
- Cilium-agent instability.

---

## Phase 7 — Runtime detection

Deploy Falco and/or Tetragon observation with:

- Representative rules.
- Event-rate measurements.
- Drop monitoring.
- SIEM integration.
- On-call runbooks.
- Noise and false-positive review.

Do not proceed to prevention when the organization cannot reliably distinguish expected from malicious behavior.

---

## Phase 8 — Runtime enforcement canary

Start with one high-confidence invariant and one noncritical workload.

Require:

- Monitor-mode evidence.
- Security approval.
- Workload-owner approval.
- Kernel compatibility.
- Automated rollback.
- Maximum affected workload count.
- Exclusion for platform recovery components.
- Post-enforcement synthetic test.

---

## Phase 9 — Expand by risk tier

Possible tiers:

- Tier A: observation only.
- Tier B: network default deny and runtime alerting.
- Tier C: selected inline prevention.
- Tier D: dedicated nodes, strict MAC/seccomp, tightly scoped network and runtime policy.

Not every workload requires the strongest enforcement. Complexity must match risk.

---

# 20. Rollback strategy

A dataplane rollback is not simply uninstalling Cilium.

Plan for:

- Existing CNI state.
- kube-proxy mode.
- Node replacement.
- iptables or BPF cleanup.
- Pod recreation.
- Service routing.
- Network-policy semantics.
- Multi-cluster connectivity.
- Encryption state.

The safest rollback often uses immutable node pools:

```text
new Cilium node pool fails acceptance
        |
        v
stop scheduling to canary
        |
        v
shift workloads to known-good node pool
        |
        v
preserve failed nodes for evidence
        |
        v
replace or rebuild canary pool
```

For runtime policy:

- Remove or disable the specific policy.
- Confirm sensors detach or the intended policy domain changes.
- Verify affected workloads recover.
- Replace nodes if persistent enforcement cannot be safely cleared.

---

# 21. Acceptance criteria

Exact targets are environment-specific, but categories must be explicit.

## Dataplane

- Zero unexplained packet-loss increase.
- DNS P99 within target.
- TCP connection success unchanged or improved.
- No unacceptable retransmission increase.
- Pod startup and endpoint regeneration within SLO.
- Cilium-agent and operator availability within target.
- No BPF program-load failure on supported AMIs.
- BPF-map pressure below defined thresholds during peak and fault tests.
- Policy convergence within target.

## Security policy

- Required flows succeed.
- Prohibited flows are denied.
- Denied flows identify source, destination, and policy context.
- No broad platform-service outage.
- Break-glass paths are controlled and audited.

## Runtime detection

- Event pipeline coverage for every target node.
- Dropped-event rate below the defined threshold.
- Alert delivery latency within target.
- False-positive rate acceptable to responders.
- Parent/child and workload context available.

## Runtime enforcement

- Protected operation is blocked at the intended hook.
- Alternate operation paths are tested.
- Enforcement scope matches the intended workload set.
- No unintended platform process termination.
- Rollback completes within the required time.
- Maximum simultaneous affected workloads is bounded.
- User SLI remains within budget during enforcement tests.

Strong close:

> The project is not complete when Cilium is installed or a Tetragon policy is accepted by the API. It is complete when required traffic remains correct under load, prohibited behavior is measurably blocked, evidence loss is visible, and a bad policy can be contained and rolled back without sacrificing the fleet.

---

# 22. Trade-off table

| Decision | Benefit | Cost or risk |
|---|---|---|
| Kubernetes NetworkPolicy | Portable, simpler semantics | Limited advanced controls and implementation differences |
| CiliumNetworkPolicy | Identity, DNS, entities, selected L7 | Cilium-specific coupling and greater policy complexity |
| Default deny | Strong lateral-movement reduction | Dependency inventory and rollout risk |
| XDP enforcement | Very early, high-rate filtering | Less context and early-drop observability challenges |
| tc datapath | Richer network context | Later in path and still kernel-critical |
| Socket-level acceleration | Efficient local/service paths | More complex path-specific debugging |
| L7 policy | Method/path/protocol control | Envoy cost, latency, and failure modes |
| Hubble | High-fidelity network evidence | Not process legitimacy or complete business proof |
| Falco | Mature behavioral rule ecosystem | Asynchronous processing and event-loss risk |
| Tetragon observation | Kernel and Kubernetes-aware context | Low-level policy expertise required |
| Tetragon override | Can prevent supported operation | Hook/kernel limitations and application impact |
| Tetragon `SIGKILL` | Immediate actor termination | Triggering operation may already have side effects |
| Persistent enforcement | Selected policy survives userspace-agent exit | Events unavailable while agent is down; recovery complexity |
| One global runtime policy | Simple central intent | Catastrophic fleet-wide false-positive risk |
| Risk-tiered enforcement | Bounded complexity and blast radius | More governance and classification work |

---

# 23. Answers that sound correct but fail at scale

## “eBPF is always faster than iptables.”

Too broad. Performance depends on workload, packet size, connection pattern, map contention, encryption, proxying, kernel, and configuration. Benchmark the actual path.

## “Cilium versus Calico means eBPF versus iptables.”

False dichotomy. Calico also has an eBPF dataplane. Compare actual modes, policy, operations, cloud integration, and migration requirements.

## “All Cilium policy executes inside eBPF.”

Incorrect for selected L7 behavior, which can redirect traffic to Envoy.

## “NetworkPolicy stops a compromised application.”

It limits reachable destinations. It does not identify malicious intent over an allowed path.

## “Falco prevents malicious syscalls.”

Falco is generally a detection engine. External response actions can react, but that is not the same as inline prevention.

## “Falco saw no alert, so no suspicious syscall occurred.”

Invalid when event drops, output failure, missing rules, or agent coverage gaps exist.

## “Tetragon kills the process before the syscall executes.”

Too absolute. `SIGKILL` terminates synchronously, but the triggering operation may already have happened. Use the correct hook and override when prevention is required.

## “The verifier means eBPF cannot crash or slow the system.”

The verifier provides strong safety checks, but kernel bugs, logical policy errors, map pressure, contention, and expensive programs remain possible.

## “We can turn on default deny by observing traffic for one day.”

Rare jobs, DR, certificate rotation, maintenance, and incident paths will be missed.

## “A cluster-wide prevention policy is easier to manage.”

It may be easier to express and far more dangerous to operate. Scope and progressive enforcement are mandatory.

---

# 24. Whiteboard exercise

Draw this from memory:

```text
Application process
   |
   | system calls / sockets
   v
Linux kernel
   |
   +--> seccomp / capabilities / LSM controls
   |
   +--> Tetragon runtime hooks
   |       +--> observe
   |       +--> override supported operation
   |       +--> signal process
   |
   +--> socket/cgroup hooks
   |
   +--> Cilium endpoint policy
   |       +--> allow
   |       +--> deny
   |       +--> redirect to Envoy for selected L7
   |
   +--> tc / routing / service LB / encryption
   |
   +--> XDP on receive where configured
   |
   v
Network

Hubble  <- datapath flow evidence
Falco   <- kernel/plugin event detection
SIEM    <- alerts and correlation
```

Then explain:

- Which controls are preventive before runtime monitoring?
- What Cilium knows that Falco does not?
- What Falco/Tetragon know that NetworkPolicy does not?
- Where events can be lost?
- Why `SIGKILL` is not equivalent to return-value override?
- What happens when Cilium agent dies but BPF programs remain loaded?
- How a bad policy can affect the entire node?

---

# 25. Adversarial interviewer follow-ups

## Why use Cilium instead of standard NetworkPolicy only?

Use standard NetworkPolicy where it is sufficient. Choose Cilium-specific policy for identity, DNS, entity, host, multi-cluster, or selected L7 capabilities that materially reduce risk. Avoid platform-specific complexity without a requirement.

## Why not use L7 policy everywhere?

It introduces userspace proxying, protocol parsing, resource cost, latency, certificate concerns, and additional failure modes. Application authorization may be stronger and more context-aware.

## Does Cilium eliminate kube-proxy?

It can replace kube-proxy when configured in that mode, but it does not have to. Treat kube-proxy replacement as a separate tested migration decision.

## What happens when the Cilium agent is down?

Already-loaded programs and state may continue forwarding existing traffic, but endpoint setup, policy updates, identity synchronization, service changes, and new pod lifecycle degrade. Define a bounded stale-control interval and node-repair policy.

## What happens if a BPF map fills?

New inserts can fail, potentially affecting connections, NAT, policy, or services depending on the map. Monitor pressure and failed operations, identify the source of growth, and avoid treating larger maps as the only fix.

## Why use Falco if Tetragon can enforce?

Detection and prevention have different goals. Falco has a mature behavioral rule ecosystem and plugin model. Tetragon offers rich kernel-aware tracing and selected inline enforcement. Organizations may use one or both based on coverage, expertise, and response design.

## Can Tetragon prevent every malicious syscall?

No. Coverage depends on hook choice, alternate kernel paths, supported actions, kernel configuration, and policy correctness. Preventive Linux controls such as seccomp and LSM profiles remain important.

## Why not kill every shell in production?

Some workloads or support paths may legitimately use interpreters. More importantly, a broad rule can kill platform recovery tools. Remove shells from images, apply preventive controls, observe real behavior, and scope enforcement to high-confidence invariants.

## What if runtime events are dropped during an attack?

Declare evidence incomplete, use independent data sources, contain the workload, preserve node and cloud audit evidence, and treat drop rate as a security incident. Do not claim no malicious activity occurred.

## How do you stop the security controller from causing an outage?

Use node/workload selectors, canary enforcement, maximum affected-workload thresholds, stop conditions, platform-component exclusions, out-of-band rollback, and immutable node replacement.

---

# 26. Hands-on lab

## Objective

Build and validate layered network and runtime controls while measuring evidence loss, policy behavior, and rollback.

## Prerequisites

- Disposable Kubernetes cluster.
- Supported Linux kernel.
- Cilium and Hubble.
- Optional Falco.
- Tetragon.
- `kubectl`, Cilium CLI, Hubble CLI, `bpftool`, and `tetra`.

Do not run enforcement experiments on production.

---

## Stage 1 — Inspect kernel capabilities

```bash
uname -a
bpftool feature probe kernel
bpftool prog show
bpftool map show
```

Record:

- Kernel version.
- BTF support.
- Relevant program types.
- Relevant helpers.
- Cgroup mode.
- JIT state.

---

## Stage 2 — Validate Cilium

```bash
cilium status --verbose
cilium connectivity test
```

Record:

- Agent health.
- Operator health.
- Hubble health.
- Connectivity failures.
- Test latency.

---

## Stage 3 — Observe baseline flows

Deploy:

- `frontend`.
- `backend`.
- `unrelated`.

Use:

```bash
hubble observe --follow
```

Identify:

- DNS queries.
- Frontend-to-backend traffic.
- Unrelated traffic.
- Source and destination identities.

---

## Stage 4 — Apply default-deny and explicit allow

Create policy that permits:

- Frontend to backend on one port.
- DNS resolution.

Verify:

- Required path succeeds.
- Unrelated path fails.
- Hubble exposes the deny reason.
- Endpoint policy revisions converge.

---

## Stage 5 — Add selected L7 policy

Allow only one HTTP method/path.

Measure:

- Added proxy components.
- P50 and P99 latency.
- Envoy resource usage.
- Behavior for denied paths.
- Failure behavior when the proxy is restarted.

---

## Stage 6 — Runtime observation

Generate controlled events:

- Spawn a shell.
- Read a test file.
- Attempt an outbound connection.
- Change process credentials in a disposable container where safe.

Observe with:

```bash
tetra getevents -o compact
```

And Falco if installed.

Compare:

- Process context.
- Parent process.
- Pod and namespace.
- Event latency.
- Rule outcome.

---

## Stage 7 — Event-pressure experiment

Generate high but safe event volume in the lab.

Observe:

- Falco dropped events.
- Tetragon event export.
- Agent CPU.
- Buffer metrics.
- Alert latency.

Do not merely increase buffers. Determine the relationship between event rate, rule complexity, CPU, and loss.

---

## Stage 8 — Canary enforcement

Choose one harmless test invariant, such as blocking access to a disposable test file or operation using an official Tetragon example.

Requirements:

- One namespace.
- One node or workload selector.
- Known rollback command.
- No platform components.
- Synthetic verification.

Test:

- Monitor behavior.
- Enforcement action.
- Application error or signal.
- Event output.
- Policy removal.
- Workload recovery.

Explain whether the action prevented the operation or only killed the process.

---

## Stage 9 — Agent interruption

In the disposable lab:

- Restart Cilium agent and observe existing versus new traffic.
- Restart Hubble Relay and observe evidence gaps.
- Restart Falco and Tetragon and observe detection/enforcement behavior.
- Test persistent enforcement only if intentionally configured.

Document:

- What continued.
- What became stale.
- What evidence was lost.
- What recovery required.

---

## Stage 10 — Rollback

Remove canary policy, drain the canary node if needed, and return workloads to known-good nodes.

Prove:

- Connectivity restored.
- No stale BPF policy remains.
- Runtime enforcement removed.
- Monitoring coverage restored.
- No unexplained process restarts remain.

---

# 27. Practice drills

## Easy

Explain the difference between XDP and tc.

## Medium

Explain eBPF maps, verifier, helpers, BTF, and CO-RE without notes.

## Hard

Explain why an allowed Cilium network connection can still represent a malicious action.

## Staff-level

Design a default-deny migration for 500 namespaces without breaking rare operational traffic.

## Principal-level

Design a runtime-security platform for:

- 5,000 nodes.
- Multiple kernel versions.
- High-volume CI pools.
- Sensitive payment workloads.
- A service mesh.
- Strict availability requirements.
- A security operations team with a five-minute response objective.

Include:

- Threat model.
- Network policy.
- Runtime detection.
- Inline prevention boundaries.
- Event-loss SLO.
- Kernel compatibility.
- Canary and rollback.
- Fleet-level blast-radius controls.
- Incident response.

---

# 28. Personal production-story mapping

Build one story from your own Linux, Kubernetes, networking, or security experience.

## Situation

A host, network, or security control needed stronger visibility or enforcement.

## Constraint

Examples:

- Production availability.
- Legacy kernel.
- Large node fleet.
- Multiple teams.
- Incomplete dependency inventory.
- High traffic.
- Limited maintenance windows.

## Evidence

Examples:

- Packet capture.
- iptables rules.
- conntrack usage.
- process tree.
- system calls.
- kernel logs.
- policy verdicts.
- cloud audit logs.

## Decision

Explain why you chose:

- prevention versus detection.
- network versus process control.
- canary versus broad rollout.
- repair versus replacement.

## Result

Use defensible metrics only.

## Learning

Tie the story to the chapter:

> The important lesson was that observability and enforcement are separate control loops. A system that can detect an event is not automatically capable of preventing it, and the enforcement mechanism itself requires a bounded failure domain.

---

# 29. Self-scoring rubric

Score each dimension from 0 to 2.

| Dimension | 0 | 1 | 2 |
|---|---|---|---|
| Linux foundations | Says “eBPF is kernel magic” | Names hooks | Explains hook context and packet/syscall path |
| eBPF mechanics | Vague | Mentions verifier/maps | Explains loader, verifier, maps, BTF, compatibility |
| Cilium | Calls it only a CNI | Explains policy | Explains identity, datapath, service LB, L7 redirect |
| Security boundaries | Mixes all tools | Separates network/runtime | Maps threats to preventive, detection, and enforcement controls |
| Falco | Calls it prevention | Calls it detection | Explains event loss and pipeline health |
| Tetragon | Says “kills before exec” | Mentions enforcement | Distinguishes override, signal, hook semantics, and bypass risk |
| Failure analysis | Ignores tool failure | Mentions agent failure | Explains map pressure, verifier, node blast radius, evidence gaps |
| Rollout | Big-bang | Canary | Kernel matrix, policy equivalence, stop criteria, rollback |
| Operations | Tool-name dump | Gives commands | Connects each command to a hypothesis and proof |
| Communication | Unstructured | Correct but long | Concise opening with deep, adversarial follow-through |

Target:

```text
17/20 or higher
```

---

# 30. Final memorization card

Do not memorize the full chapter. Memorize this chain:

```text
Linux process and packet path
 -> choose the correct hook
 -> verifier loads constrained program
 -> maps hold dataplane/security state
 -> Cilium identity controls network reachability
 -> Hubble explains flow verdicts
 -> NetworkPolicy cannot identify malicious intent over an allowed path
 -> Falco detects but event buffers can drop evidence
 -> Tetragon can filter and enforce in kernel
 -> Override can prevent supported operation
 -> SIGKILL terminates but may not undo side effect
 -> canary by kernel, node, namespace, and workload
 -> monitor map pressure, event loss, policy convergence, and customer SLI
 -> bound the security system’s own blast radius
```

Final Principal-level sentence:

> I use eBPF only after defining the exact security invariant and the kernel hook that can enforce it. Cilium controls reachable network paths, Hubble provides dataplane evidence, Falco detects behavior with explicit event-loss monitoring, and Tetragon enforces only high-confidence operations with bounded scope and proven rollback. The security platform must remain observable and survivable under the same catastrophic stress it is intended to defend against.

---

# 31. Official primary references

- Linux kernel — BPF documentation:  
  https://docs.kernel.org/bpf/
- Linux kernel — BPF Type Format:  
  https://docs.kernel.org/bpf/btf.html
- Linux kernel — libbpf and CO-RE overview:  
  https://docs.kernel.org/bpf/libbpf/libbpf_overview.html
- Cilium — Introduction to Cilium and Hubble:  
  https://docs.cilium.io/en/stable/overview/intro/
- Cilium — eBPF datapath introduction:  
  https://docs.cilium.io/en/stable/network/ebpf/intro/
- Cilium — BPF and XDP reference guide:  
  https://docs.cilium.io/en/stable/reference-guides/bpf/
- Cilium — Network-policy overview:  
  https://docs.cilium.io/en/stable/security/policy/
- Cilium — Security identities:  
  https://docs.cilium.io/en/stable/internals/security-identities/
- Cilium — Hubble internals:  
  https://docs.cilium.io/en/stable/internals/hubble/
- Cilium — Troubleshooting:  
  https://docs.cilium.io/en/stable/operations/troubleshooting/
- Cilium — Metrics and BPF-map pressure:  
  https://docs.cilium.io/en/stable/observability/metrics/
- Falco — Dropped syscall events:  
  https://falco.org/docs/concepts/event-sources/kernel/dropped-events/
- Falco — Troubleshooting dropped events:  
  https://falco.org/docs/troubleshooting/dropping/
- Tetragon — TracingPolicy:  
  https://tetragon.io/docs/concepts/tracing-policy/
- Tetragon — Enforcement concepts:  
  https://tetragon.io/docs/concepts/enforcement/
- Tetragon — Policy enforcement guide:  
  https://tetragon.io/docs/getting-started/enforcement/
- Tetragon — Selectors and actions:  
  https://tetragon.io/docs/concepts/tracing-policy/selectors/
- Tetragon — Persistent enforcement:  
  https://tetragon.io/docs/concepts/enforcement/persistent-enforcement/
