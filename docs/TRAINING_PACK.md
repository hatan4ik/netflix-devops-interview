**NETFLIX-SCALE**

DevOps / Platform Engineering  
Interview Training Pack

17 advanced systems, Kubernetes, cloud, Linux, RCA, chaos, and
leadership scenarios

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th><p><strong>Prepared for Nathanel Sulimanov<br />
Senior / Principal DevOps, Platform and Cloud Architecture</strong></p>
<p>Interview-ready answers, deep technical explanations, commands,
failure trees, follow-up questions, and a 7-day practice plan</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

*Verified against current official documentation - July 22, 2026*

Important: These are hypothetical interview scenarios. They are not a
claim about Netflix's private production architecture or actual
interview process.

# How to Use This Pack

This pack is designed for a senior or principal-level interview. The
goal is not to memorize product names. The goal is to demonstrate a
repeatable engineering method: define the objective, identify the
failure domain, use evidence to reduce uncertainty, restore service
safely, and then remove the class of failure.

## The two answer frameworks

### Architecture question: SCOPE

- **S - State assumptions and goals.** Traffic volume, latency
  objective, consistency requirement, tenancy, compliance, RTO, and RPO.

- **C - Control plane and data plane.** Explain who calculates desired
  state and who serves traffic when the control plane is unavailable.

- **O - Operability.** Metrics, logs, traces, rollout, rollback,
  capacity, and failure isolation.

- **P - Protection.** Identity, least privilege, encryption, policy,
  rate limits, and blast-radius controls.

- **E - Economics and trade-offs.** Complexity, cost, latency,
  portability, and organizational ownership.

### Incident question: STABILIZE

- **S - Stop the blast radius.** Pause rollout, freeze conflicting
  changes, and establish an incident commander.

- **T - Trace the user path.** Follow one failing request through every
  hop rather than staring at dashboards in isolation.

- **A - Ask what changed.** Config, certificates, dependency behavior,
  traffic shape, quotas, and data distribution.

- **B - Bound the failure.** Region, zone, node group, workload version,
  tenant, protocol, or request class.

- **I - Inspect authoritative signals.** Control-plane status,
  data-plane config, kernel/network evidence, and application telemetry.

- **L - Limit harm.** Roll back, route around, reduce concurrency,
  disable a noncritical feature, or fail open/closed deliberately.

- **I - Identify root and contributing causes.** The trigger is not
  always the systemic cause.

- **Z - Zero out recurrence.** Add tests, guardrails, observability,
  ownership, and automated rollback.

- **E - Explain impact and learning.** Customer impact, timeline,
  decision quality, and measurable prevention.

| **Senior-level signal:** Say what you would \*not\* do. Examples: do not force-unlock Terraform state until ownership is proven; do not make liveness depend on a remote database; do not restart every pod before collecting evidence; do not declare a region healthy based only on load-balancer health checks. |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

## A strong 90-second answer pattern

1\. Open with the objective and one or two assumptions.

2\. Describe the architecture or triage path in layers.

3\. Name the authoritative commands or metrics you would use.

4\. Give the safest immediate mitigation.

5\. Close with prevention, rollout, and trade-offs.

# Round 1 - Systems at Scale, Kubernetes, Cloud, and Linux

## 1. Fine-grained service discovery across 1,000+ microservices with Envoy or Istio

### What the interviewer is testing

Whether you understand service discovery as a control-plane scaling
problem, not merely DNS. At 1,000+ services, the risk is that every
proxy receives every endpoint, route, policy, and certificate. That
creates large xDS payloads, high proxy memory, slow convergence, and a
dangerous mesh-wide blast radius.

### Interview-ready answer

I would separate the registry, the mesh control plane, and the proxy
data plane. Kubernetes or another registry remains the source of service
and endpoint truth. Istiod or a custom xDS management plane converts
that into scoped CDS, EDS, RDS, LDS, and SDS resources for Envoy. The
critical design rule is that a workload receives only the services and
routes it can actually call.

I would partition by trust domain, environment, region, and business
domain; use namespace or workload selectors; restrict service visibility
with exportTo and Istio Sidecar egress hosts; and use ServiceEntry or
WorkloadEntry for non-Kubernetes endpoints. For multi-cluster, I would
prefer local endpoints, fail over by locality, and avoid creating a
single global endpoint set unless the application truly requires it.

Operationally, I would measure xDS push time, config size per proxy,
NACKs, convergence time, proxy memory, endpoint churn, and stale
configuration. Changes roll out through canary control-plane revisions
and progressive namespace onboarding. The data plane must continue
serving its last accepted configuration if the control plane is
temporarily unavailable.

### Deep design

- **Registry layer:** Kubernetes Services and EndpointSlices, cloud
  registries, VM registries, or a dedicated catalog. Normalize service
  identity and ownership metadata.

- **Control-plane layer:** Compute the smallest configuration graph per
  workload. Use incremental or delta xDS where appropriate, avoid full
  snapshots for small endpoint changes, and horizontally scale discovery
  servers by shard.

- **Data-plane layer:** Envoy performs client-side load balancing,
  health awareness, circuit breaking, retries, and telemetry using the
  last ACKed config.

- **Visibility boundaries:** A payments workload should not receive
  routes for studio rendering, HR, or unrelated test namespaces. This is
  both a security and scale control.

- **Locality:** Prefer same-zone or same-region endpoints, but maintain
  enough cross-zone capacity to survive failure. Use outlier detection
  carefully so correlated failures do not eject every endpoint.

- **Naming:** Use stable service identity independent of deployment
  name. Version through subsets and routing rules, not by forcing
  clients to learn new names.

### Concrete configuration example

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th>apiVersion: networking.istio.io/v1<br />
kind: Sidecar<br />
metadata:<br />
name: payments-scope<br />
namespace: payments<br />
spec:<br />
egress:<br />
- hosts:<br />
- "./*"<br />
- "identity/*"<br />
- "ledger/*"<br />
- "istio-system/*"<br />
---<br />
apiVersion: networking.istio.io/v1<br />
kind: DestinationRule<br />
metadata:<br />
name: ledger<br />
namespace: payments<br />
spec:<br />
host: ledger.ledger.svc.cluster.local<br />
trafficPolicy:<br />
outlierDetection:<br />
consecutive5xxErrors: 5<br />
interval: 5s<br />
baseEjectionTime: 30s<br />
maxEjectionPercent: 20<br />
loadBalancer:<br />
localityLbSetting:<br />
enabled: true</th>
</tr>
</thead>
<tbody>
</tbody>
</table>

### Verification and debugging

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th>istioctl proxy-status<br />
istioctl proxy-config clusters &lt;pod&gt; -n payments<br />
istioctl proxy-config endpoints &lt;pod&gt; -n payments \<br />
--cluster 'outbound|443||ledger.ledger.svc.cluster.local'<br />
istioctl proxy-config routes &lt;pod&gt; -n payments<br />
kubectl get endpointslice -A</th>
</tr>
</thead>
<tbody>
</tbody>
</table>

### Trade-offs and pitfalls

- One giant mesh simplifies identity naming but increases config and
  organizational blast radius.

- Too many independent meshes reduce blast radius but add cross-mesh
  gateways, certificate federation, and debugging complexity.

- Retries and outlier detection can improve availability but can amplify
  overload if budgets are not bounded.

- DNS-only discovery is simple but lacks rich identity, policy, route,
  and endpoint-health semantics.

### Strong closing line

My success criterion is not that Istiod knows about 1,000 services. It
is that each proxy receives a small, correct, quickly converging view of
the services it is authorized to reach, and that a discovery-plane
outage does not interrupt established traffic. \[1\]\[2\]\[3\]\[4\]

### Likely follow-ups

- **How do you handle config explosion?** Scope visibility, shard
  control planes, use incremental delivery, reduce route duplication,
  and measure bytes and resources per proxy.

- **How do you roll out mesh upgrades?** Revisioned control planes,
  canary namespaces, compatibility tests, config analysis, and explicit
  rollback criteria.

## 2. eBPF and Cilium for runtime network security

### What the interviewer is testing

Whether you understand where policy is enforced, how identity replaces
fragile IP rules, and why observability must accompany enforcement.

### Interview-ready answer

I would use Cilium to attach eBPF programs at the pod and node datapath,
assign security identities from Kubernetes labels, and enforce
default-deny policies at L3/L4 with selective L7 rules where the
protocol is understood. I would start in observe mode using Hubble,
build a flow inventory, generate proposed policies, test for
denied-but-required traffic, and then enforce progressively by namespace
and workload tier.

Compared with traditional iptables-heavy CNIs, eBPF can enforce closer
to the packet path, avoid long rule chains, support identity-aware
policy despite changing pod IPs, and provide high-fidelity flow
verdicts. Cilium can also combine network policy, load balancing, and
visibility in one datapath. I would still treat kernel compatibility,
map sizing, policy complexity, and operational expertise as first-class
risks.

### Policy model

- **Identity first:** Select workloads by labels and service accounts,
  not ephemeral IP addresses.

- **Default deny:** Separate ingress and egress. Permit only required
  destinations, ports, and protocols.

- **L7 selectively:** HTTP method/path or DNS-aware rules can be
  powerful, but they add parsing and operational complexity. Do not
  place every packet through L7 logic without a reason.

- **Host and node policy:** Protect metadata endpoints, kubelet,
  control-plane access, and node-local services.

- **FQDN policy:** Useful for controlled egress, while recognizing DNS
  TTL, CDN address changes, and fail-closed behavior.

- **Visibility:** Hubble flow records, policy verdicts, dropped packets,
  DNS queries, TCP resets, and latency become part of the incident
  toolchain.

### Example policy

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th>apiVersion: cilium.io/v2<br />
kind: CiliumNetworkPolicy<br />
metadata:<br />
name: playback-api<br />
namespace: playback<br />
spec:<br />
endpointSelector:<br />
matchLabels:<br />
app: playback-api<br />
ingress:<br />
- fromEndpoints:<br />
- matchLabels:<br />
io.kubernetes.pod.namespace: edge<br />
app: edge-gateway<br />
toPorts:<br />
- ports:<br />
- port: "8443"<br />
protocol: TCP<br />
rules:<br />
http:<br />
- method: "GET"<br />
path: "^/v1/(manifest|segment)/.*"<br />
egress:<br />
- toEndpoints:<br />
- matchLabels:<br />
io.kubernetes.pod.namespace: metadata<br />
app: title-metadata<br />
toPorts:<br />
- ports:<br />
- port: "443"<br />
protocol: TCP</th>
</tr>
</thead>
<tbody>
</tbody>
</table>

### Safe rollout

1\. Baseline Hubble flows for at least a representative traffic cycle.

2\. Identify cron, backup, failover, and rarely used administrative
paths.

3\. Apply policy to a canary workload or namespace.

4\. Alert on denies and SLO impact before broad enforcement.

5\. Keep a break-glass process with time-limited approval and audit.

6\. Use policy tests in CI so application changes declare new network
dependencies.

### Debugging commands

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th>cilium status --verbose<br />
cilium connectivity test<br />
hubble observe --namespace playback --verdict DROPPED<br />
hubble observe --from-pod playback/playback-api --follow<br />
kubectl get cnp,ccnp -A<br />
cilium bpf policy get<br />
cilium monitor --type drop</th>
</tr>
</thead>
<tbody>
</tbody>
</table>

### Advantages over traditional CNIs

- Identity and policy remain stable while pod IPs churn.

- Rule lookup uses eBPF maps rather than traversing very large iptables
  chains.

- Rich flow visibility is generated from the same datapath that enforces
  policy.

- Socket- or service-level load balancing can reduce extra hops in some
  designs.

- Policies can extend beyond basic Kubernetes NetworkPolicy where
  justified.

### Risks

- Kernel and Cilium version compatibility must be tested on every node
  image.

- A powerful datapath increases the impact of a bad upgrade; use canary
  node groups.

- L7 policy may depend on proxying and protocol correctness.

- Do not claim eBPF is automatically faster in every workload; benchmark
  your packet size, connection pattern, and encryption mode.

### Strong closing line

I would sell Cilium not as a fashionable CNI, but as an identity-aware
enforcement and evidence layer. The migration is successful only when we
can explain every allowed path, detect every denied path, and roll back
a datapath change without a fleet-wide outage. \[6\]\[7\]\[8\]

## 3. Multi-cloud routing, IAM, and secret synchronization

### Interview-ready answer

I would avoid pretending that three clouds are one network. Each cloud
remains an independent failure and security domain, connected through a
small number of well-defined transit and service edges. Routing uses
redundant private connectivity or encrypted tunnels, non-overlapping
address plans, explicit route ownership, and application-level failover.
Identity uses federation and short-lived workload credentials;
long-lived cross-cloud access keys are prohibited. Secrets have one
authoritative owner and are distributed or retrieved through audited
automation, not copied manually between clusters.

For routing, I would define whether traffic is user ingress,
service-to-service, control plane, or data replication. User traffic
goes through a global traffic layer. East-west traffic crosses gateways
with mTLS and explicit service identity. Data replication has separate
bandwidth, consistency, and failure controls. I would prefer local reads
and local dependencies; cross-cloud synchronous calls are exceptional
because they combine latency and correlated failure.

### Routing design

- Non-overlapping CIDRs and a centrally governed IPAM plan.

- Redundant cloud routers or transit hubs in each environment.

- BGP with route filters, maximum-prefix protection, and clear
  ownership.

- Service gateways instead of full network reachability where possible.

- Locality-aware routing and circuit breakers to keep a remote-cloud
  failure from exhausting local resources.

- Dedicated paths and quotas for replication, backup, CI/CD, and
  observability.

- Continuous synthetic probes from every cloud to every critical
  endpoint.

### IAM design

- Humans authenticate through a corporate identity provider with MFA and
  just-in-time elevation.

- Workloads use Kubernetes service account federation: EKS Pod Identity
  or IRSA, Microsoft Entra Workload ID, and Google Workload Identity
  Federation.

- Cross-cloud access is a token exchange with audience and subject
  restrictions, not a stored access key.

- Roles are scoped by namespace, service account, environment, and
  resource tags.

- Cloud break-glass roles are isolated, hardware-protected, monitored,
  and time-limited.

### Secret strategy

Choose one of two patterns deliberately:

1\. **Runtime retrieval:** The application or sidecar retrieves a
short-lived secret from a central system such as Vault. This minimizes
copies but creates runtime dependency and latency considerations.

2\. **Controlled synchronization:** An operator synchronizes selected
secrets into each cloud-native secret manager. This improves local
availability but creates multiple encrypted copies and requires strict
version, rotation, and deletion semantics.

For both patterns, every secret needs an owner, rotation period,
consumers, classification, and revocation test. Certificates and
database credentials should be short-lived where the platform supports
it. Secret values never appear in Git, Terraform plans, CI logs, or
support bundles.

### Failure scenarios I would test

- One cloud loses connectivity to the identity provider.

- Token exchange or OIDC JWKS retrieval fails.

- A secret rotates in the source but one destination misses the update.

- Clock skew invalidates tokens or certificates.

- BGP advertises an overly broad prefix.

- Cross-cloud latency increases and retries create a traffic storm.

### Strong closing line

My design goal is independent survivability: each cloud can continue its
critical local service when another cloud, the interconnect, or the
central secret system is impaired. Federation replaces stored
credentials, and cross-cloud calls are explicit exceptions with budgets,
not invisible dependencies. \[24\]\[25\]

## 4. Intermittent systemd unit failures on EKS nodes: detect and heal

### Interview-ready answer

First I would classify the unit: kubelet, containerd, networking,
storage, security agent, or a custom daemon. I would detect the failure
from both the host and Kubernetes control plane: systemctl show,
journald, node conditions, kubelet events, container runtime metrics,
and workload symptoms. For AWS-managed signals, I would use the EKS node
monitoring agent and node repair; for custom units, Node Problem
Detector can publish a custom node condition or event.

Healing depends on the failure class. A safe, one-time unit restart may
be appropriate for a known transient failure. Repeated failure or a
critical runtime problem should cordon and drain the node, preserve
logs, and replace the instance from an immutable node group. I would not
build an endless systemctl restart loop that hides a corrupt image,
kernel problem, disk issue, or bad rollout.

### Triage sequence

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th>kubectl get nodes -o wide<br />
kubectl describe node &lt;node&gt;<br />
kubectl get events --all-namespaces --field-selector
involvedObject.kind=Node \<br />
--sort-by=.lastTimestamp<br />
<br />
systemctl --failed<br />
systemctl show kubelet -p ActiveState -p SubState -p Result -p
NRestarts<br />
journalctl -u kubelet --since '-30 min' --no-pager<br />
journalctl -u containerd --since '-30 min' --no-pager<br />
dmesg -T | tail -200<br />
crictl info<br />
crictl ps -a</th>
</tr>
</thead>
<tbody>
</tbody>
</table>

### Detection layers

- CloudWatch or another log pipeline for journald patterns and unit
  state.

- Node Problem Detector custom plugin for specific unit checks.

- EKS node monitoring agent for supported node-health categories.

- Prometheus alerts on node readiness, kubelet scrape failure, runtime
  operations, filesystem, PID pressure, and restart frequency.

- Correlation with an AMI version, kernel version, instance type,
  availability zone, or recent node-group update.

### Remediation state machine

1\. **Transient and understood:** Restart the service once, verify
health, and open a ticket if recurrence exceeds threshold.

2\. **Critical or repeated:** Taint or cordon the node so new work does
not land there.

3\. **Protect workloads:** Drain while honoring disruption budgets;
verify stateful and local-volume implications.

4\. **Capture evidence:** Journald, dmesg, runtime state, disk health,
network state, process limits, and instance console output.

5\. **Replace:** Terminate through the managed node group, Karpenter, or
EKS node repair rather than repairing an unknown snowflake indefinitely.

6\. **Prevent:** Fix the image or bootstrap code, canary a new node
group, and automate validation.

### Common hidden causes

- Disk full or inode exhaustion causing journald, containerd, or kubelet
  failure.

- OOM kill or PID exhaustion.

- Expired certificates, clock skew, or failed DNS.

- CNI initialization ordering or iptables/eBPF state conflict.

- Invalid systemd dependency or restart throttling (StartLimitBurst).

- Kernel regression or unsupported module on a custom AMI.

- Node bootstrap completes partially and the unit later loses
  credentials or config.

### Strong closing line

A node is cattle, but the evidence on a failing node is valuable. I
collect enough evidence to identify the class of failure, then replace
the node safely. The durable fix belongs in the immutable image,
bootstrap pipeline, or managed add-on - not in an SSH session.
\[9\]\[10\]

## 5. Pre-production vetting of custom EKS AMIs at kernel and runtime

### Interview-ready answer

I would treat an AMI as a versioned product with source, build recipe,
SBOM, tests, signing or provenance, and a promotion path. Teams do not
hand me an AMI ID and ask for production access. They contribute
components to an Image Builder or Packer pipeline based on a known
EKS-optimized image, and the pipeline proves kernel, container runtime,
kubelet, CNI, storage, security, and workload compatibility.

The image passes static security scanning, boot tests, node-join tests,
Kubernetes conformance-like smoke tests, Cilium or CNI connectivity
tests, storage attach/mount, DNS, image pull, drain, reboot, and fault
tests. I then launch it into a canary node group with taints, schedule
synthetic workloads, compare performance and error rates, and promote by
immutable launch-template version.

### Build controls

- Pin the parent image by digest or approved SSM parameter resolution.

- Record packages, kernel, modules, sysctls, systemd units, containerd
  config, and bootstrap configuration.

- Generate SBOM and vulnerability findings; define severity and
  exploitability gates.

- Use Image Builder build and test components; sign pipeline artifacts
  where available.

- Apply CIS or STIG hardening only with Kubernetes compatibility tests.
  A benchmark setting that breaks kubelet or networking is not a
  successful hardening outcome.

- Never bake application secrets or long-lived credentials into the
  image.

### Kernel and OS tests

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th>uname -a<br />
cat /etc/os-release<br />
sysctl -a | grep -E 'net.ipv4.ip_forward|rp_filter|bridge-nf'<br />
lsmod<br />
bpftool feature probe kernel<br />
systemd-analyze critical-chain<br />
systemctl --failed<br />
journalctl -p warning..alert -b</th>
</tr>
</thead>
<tbody>
</tbody>
</table>

Validate:

- Required eBPF, conntrack, overlay, encryption, filesystem, and storage
  modules.

- cgroup mode and container runtime compatibility.

- File descriptor, PID, conntrack, and ephemeral port limits.

- Time synchronization, DNS, CA bundle, proxy settings, and metadata
  protection.

- Read-only or locked-down settings do not block kubelet, CNI, CSI, or
  observability agents.

### Kubernetes runtime tests

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th>kubectl wait node/&lt;canary-node&gt; --for=condition=Ready
--timeout=10m<br />
kubectl get node &lt;canary-node&gt; -o json | jq
'.status.nodeInfo'<br />
cilium connectivity test # when Cilium is used<br />
kubectl run dns-test --image=busybox:stable --restart=Never -- nslookup
kubernetes.default<br />
kubectl drain &lt;canary-node&gt; --ignore-daemonsets
--delete-emptydir-data</th>
</tr>
</thead>
<tbody>
</tbody>
</table>

Test image pulls, service networking, DNS, NetworkPolicy, CSI volumes,
projected service-account tokens, Pod Identity, log shipping, metrics,
graceful drain, and reboot recovery. Include representative
high-throughput and latency-sensitive workloads because a node can be
functionally correct but operationally worse.

### Promotion and rollback

- dev image -\> integration node group -\> pre-production soak -\> 1%
  production canary -\> staged node-group update.

- Promotion references an immutable image digest and launch-template
  version.

- Rollback launches the last known-good image; do not mutate failed
  nodes in place.

- Retain logs, test results, SBOM, and vulnerability exception approvals
  with the image version.

### Strong closing line

Custom AMIs are acceptable when they are reproducible, observable, and
disposable. The approval artifact is not the AMI itself; it is the
evidence that the image can join, serve, fail, drain, reboot, and be
replaced without violating the service SLO. \[11\]\[12\]

## 6. Advanced Kubernetes probes for business-logic failures

### Interview-ready answer

I use startup, readiness, and liveness for different decisions. Startup
answers, "Has initialization completed?" Readiness answers, "Should this
pod receive new traffic now?" Liveness answers, "Is the process
irrecoverably stuck and likely to recover after restart?" I do not use
one deep /health endpoint for all three.

For business logic, readiness can validate local ability to serve a
request: configuration loaded, event loop responsive, required local
cache initialized, queue lag below a safety threshold, certificate
valid, and a lightweight critical-dependency check within a strict
timeout. Liveness stays shallow and local so a remote database outage
does not restart every pod. End-to-end correctness belongs in external
synthetic probes and SLO monitoring, not only kubelet probes.

### Example

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th>startupProbe:<br />
httpGet:<br />
path: /health/startup<br />
port: admin<br />
periodSeconds: 5<br />
failureThreshold: 60<br />
<br />
readinessProbe:<br />
httpGet:<br />
path: /health/ready<br />
port: admin<br />
periodSeconds: 5<br />
timeoutSeconds: 2<br />
failureThreshold: 2<br />
successThreshold: 2<br />
<br />
livenessProbe:<br />
httpGet:<br />
path: /health/live<br />
port: admin<br />
periodSeconds: 10<br />
timeoutSeconds: 1<br />
failureThreshold: 3</th>
</tr>
</thead>
<tbody>
</tbody>
</table>

### Designing the readiness contract

A playback manifest service might return not-ready when:

- Its routing/config generation is stale beyond an allowed age.

- Required signing keys or certificates expire within a safety window.

- Its worker queue is saturated and accepting more traffic would worsen
  recovery.

- A critical local state machine is inconsistent.

- It cannot reach any member of a required dependency set after a very
  small, cached check.

It should not run a heavy SQL query or create a new dependency
connection on every probe. Cache the result briefly, cap probe
concurrency, and expose the reason as a metric and log field.

### Business correctness outside kubelet

- Synthetic transactions validate the full user journey: authenticate,
  fetch metadata, create manifest, fetch a byte range, and verify
  expected content.

- Prometheus alerts measure success rate, latency, freshness, and
  correctness SLIs.

- A canary or shadow request can detect semantic failures before broad
  rollout.

- Service-mesh active health checking or outlier detection may
  complement, but not replace, application-level readiness.

### Failure-mode cautions

- A readiness failure across every pod can remove all endpoints. Use
  fail-safe thresholds and capacity headroom.

- A liveness endpoint that checks a common dependency can create a
  synchronized restart storm.

- An exec probe that leaks processes or takes locks can become the
  outage.

- Very aggressive thresholds cause flapping; very slow thresholds serve
  broken traffic too long.

- Termination must align with readiness removal, preStop, connection
  draining, and terminationGracePeriodSeconds.

### Strong closing line

The probe contract must map to the action Kubernetes will take. I only
let liveness restart a process when restart is the likely cure, and I
use readiness plus external synthetics to catch business failure without
turning a dependency outage into a restart storm. \[13\]

## 7. DNS-level outage inside a service mesh without full application redeploy

### Interview-ready answer

First I would determine which resolver path is failing: application
libc, node-local cache, CoreDNS, upstream resolver, mesh DNS proxy, or
external authoritative DNS. I would verify from an affected pod and the
sidecar, inspect CoreDNS health and saturation, and compare a working
node or namespace.

The mitigation depends on the boundary. I can scale or roll CoreDNS
without redeploying applications. If the mesh already uses DNS proxying
or ServiceEntry, I can update service discovery dynamically. For a
critical external dependency, I can create or update a ServiceEntry with
controlled static endpoints or route to a known fallback through a
VirtualService. Envoy receives xDS updates without an application
rollout. I would use static IPs only as a temporary, TTL-bound emergency
measure because certificate identity, CDN changes, and endpoint
ownership still matter.

### Triage

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th>kubectl exec -n playback &lt;pod&gt; -c app -- cat
/etc/resolv.conf<br />
kubectl exec -n playback &lt;pod&gt; -c app -- nslookup
dependency.example.com<br />
kubectl exec -n playback &lt;pod&gt; -c istio-proxy -- \<br />
pilot-agent request GET clusters | grep dependency<br />
kubectl -n kube-system get deploy,pod,svc,endpointslice -l
k8s-app=kube-dns<br />
kubectl -n kube-system logs deploy/coredns --since=15m<br />
kubectl -n kube-system top pod -l k8s-app=kube-dns<br />
istioctl proxy-config clusters &lt;pod&gt; -n playback</th>
</tr>
</thead>
<tbody>
</tbody>
</table>

### No-app-redeploy mitigation options

- Scale CoreDNS, correct an upstream forwarder, or roll only DNS
  components.

- Change a CoreDNS ConfigMap and roll CoreDNS, after validating syntax
  and rollback.

- Update ServiceEntry, WorkloadEntry, VirtualService, or
  DestinationRule; the mesh pushes new config dynamically.

- Route the dependency through an egress gateway with a stable service
  address.

- Shift traffic to a healthy region or dependency endpoint.

- Reduce DNS query load through NodeLocal DNSCache or mesh DNS proxying
  as a planned improvement.

### Emergency ServiceEntry example

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th>apiVersion: networking.istio.io/v1<br />
kind: ServiceEntry<br />
metadata:<br />
name: emergency-origin<br />
namespace: playback<br />
spec:<br />
hosts:<br />
- origin.example.com<br />
addresses:<br />
- 192.0.2.40/32<br />
ports:<br />
- number: 443<br />
name: tls<br />
protocol: TLS<br />
resolution: STATIC<br />
endpoints:<br />
- address: 192.0.2.40</th>
</tr>
</thead>
<tbody>
</tbody>
</table>

Before using this, confirm that TLS SNI and certificate validation still
use origin.example.com, that the IP is provider-approved, and that the
override expires automatically.

### Root-cause dimensions

- CoreDNS CPU throttling, memory, file descriptors, or insufficient
  replicas.

- Negative caching and TTL behavior.

- Upstream resolver rate limits or network policy blocking UDP/TCP 53.

- Search-domain expansion creating excessive queries.

- ndots behavior causing repeated suffix lookups.

- Node-local conntrack or packet loss.

- Authoritative DNS incident or bad record deployment.

### Strong closing line

I restore name-to-endpoint resolution at the smallest layer that is
broken. The mesh is useful because routing and endpoint configuration
can change dynamically, but I treat emergency static mappings as
controlled debt with an owner and expiration. \[5\]

## 8. Terraform remote-state backend timeouts: recovery and damage containment

### Interview-ready answer

The first action is to stop all Terraform writers and determine whether
the failure happened before lock acquisition, during provider changes,
or while writing state. I do not bypass locking. I preserve the CLI
output, plan file, lock ID, pipeline run, and any local emergency state
file Terraform produced.

I then restore backend connectivity and inspect the authoritative state
version, object-store version history, and lock owner. If an apply may
have partially changed resources, I inventory the cloud and run a
refresh-only plan after the backend is healthy. I use force-unlock only
when I have proven the lock is stale and no process owns it. terraform
state push is a last-resort recovery action after backups, lineage and
serial validation, peer review, and a maintenance window.

### Containment checklist

1\. Disable or pause CI jobs for the affected workspace.

2\. Announce a single incident owner and prohibit local applies.

3\. Preserve logs, .terraform metadata, plan artifact, provider
versions, and any local state fallback.

4\. Determine backend scope: one bucket/object, lock table, network
path, identity, KMS, or regional service.

5\. Verify whether provider API calls occurred and which resources
changed.

6\. Restore backend first; do not continue with -lock=false.

### Safe commands

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th>terraform version<br />
terraform providers<br />
terraform state pull &gt; state-backup-$(date +%Y%m%d-%H%M%S).json<br />
terraform plan -refresh-only -out=refresh.tfplan<br />
terraform show refresh.tfplan<br />
<br />
# Only after proving the lock is stale and owned by the failed
run:<br />
terraform force-unlock &lt;LOCK_ID&gt;</th>
</tr>
</thead>
<tbody>
</tbody>
</table>

### Recovery decision tree

- **Timeout before lock and before apply:** Fix backend/network and
  retry the plan.

- **Lock acquired, no provider changes:** Confirm run termination, then
  allow automatic unlock or carefully force-unlock stale ownership.

- **Provider changes occurred, state write uncertain:** Compare cloud
  audit logs and resources with remote state; look for a local state
  file written after backend persistence failure.

- **Remote state corrupted or deleted:** Restore an object-store version
  or backend snapshot, then perform refresh-only reconciliation.

- **State is stale but resources exist:** Prefer import/move operations
  and reviewed reconciliation over blind recreation.

### Damage-prevention architecture

- Versioning, encryption, retention, and cross-region backup for the
  state object.

- A backend that supports locking and strong access controls.

- One pipeline writer per workspace; human access is read-only by
  default.

- Network and identity monitoring for backend operations.

- Saved plan promotion so apply uses the reviewed plan and pinned
  providers.

- Smaller state domains so one backend incident does not freeze the
  entire company.

- Regular state-restore drills.

### What I would explicitly avoid

- -lock=false during an outage.

- Force-unlocking because "the pipeline looks stuck" without checking
  the owner.

- Editing JSON state manually as the first response.

- Running import or state push before taking a backup.

- Assuming a failed terraform apply means no infrastructure changed.

### Strong closing line

Terraform state is a transactional record, not just a file. My recovery
sequence protects against the two worst outcomes: concurrent writers and
infrastructure that changed without a trustworthy record. \[14\]\[15\]

# Round 2 - RCA, Fire Drills, and Netflix-Scale Chaos

## 9. Envoy rollout silently breaks mTLS between edge and mesh

### Interview-ready answer

I would pause the rollout and compare a failing proxy with a known-good
proxy. I need to know whether the failure is at certificate delivery,
listener/route selection, client TLS origination, server peer
authentication, trust validation, or protocol detection.

I would inspect control-plane sync and NACKs, then the actual data-plane
configuration: listeners, clusters, routes, endpoints, and SDS secrets.
I would compare PeerAuthentication and DestinationRule behavior, SNI,
SAN validation, trust domain, port naming, certificate expiry, and clock
skew. The immediate mitigation is to roll back or route around the new
proxy/config revision, not to weaken mesh-wide mTLS unless the security
owner explicitly accepts that emergency risk.

### RCA trace

1\. **Impact boundary:** Which edge version, region, namespace,
destination service, port, and protocol?

2\. **Change boundary:** Envoy binary, bootstrap, xDS config, TLS
policy, certificate authority, gateway secret, or workload labels?

3\. **Control plane:** Are proxies SYNCED? Are they NACKing
LDS/CDS/RDS/EDS/SDS?

4\. **Client config:** Does the edge cluster use ISTIO_MUTUAL, correct
SNI, correct SAN match, and expected endpoint port?

5\. **Server config:** Does PeerAuthentication require STRICT mTLS? Is
the workload actually in the mesh and listening on the port Envoy
expects?

6\. **Certificates:** Is SDS delivering the workload certificate and
trust bundle? Expiry, chain, trust domain, and key mismatch?

7\. **Wire evidence:** TLS alert, reset reason, upstream transport
failure, or no route?

### Commands

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th>istioctl proxy-status<br />
istioctl analyze -A<br />
istioctl proxy-config listeners &lt;edge-pod&gt; -n edge<br />
istioctl proxy-config routes &lt;edge-pod&gt; -n edge<br />
istioctl proxy-config clusters &lt;edge-pod&gt; -n edge --fqdn
playback.playback.svc.cluster.local<br />
istioctl proxy-config secret &lt;edge-pod&gt; -n edge<br />
<br />
kubectl get peerauthentication,destinationrule,virtualservice -A<br />
kubectl logs &lt;edge-pod&gt; -c istio-proxy --since=15m</th>
</tr>
</thead>
<tbody>
</tbody>
</table>

For raw Envoy, /config_dump, /clusters, /certs, and filtered /stats show
what the proxy actually accepted. Compare old and new snapshots, not
just YAML in Git.

### Common causes

- A broad DestinationRule forces plaintext while the server requires
  STRICT mTLS.

- Wrong SNI or SAN validation after service name or trust-domain change.

- Port name changed from http/grpc/tls, changing protocol detection.

- Gateway secret mounted under the wrong name or namespace.

- Certificate rotation delivered a new root to only part of the fleet.

- Clock skew makes a valid certificate appear expired or not yet valid.

- An EnvoyFilter or bootstrap change removes TLS transport socket
  configuration.

- Workload selector unexpectedly applies policy to the wrong revision.

### Mitigation and prevention

- Roll back the canary revision or shift traffic to the last known-good
  gateway pool.

- Freeze config pushes and preserve failing proxy dumps.

- Add pre-production handshake tests for edge-to-every-critical-service
  identity.

- Canary CA bundle and proxy upgrades separately.

- Gate rollout on xDS NACKs, handshake error rate, and synthetic mTLS
  transactions.

- Use policy analysis and config diffing in CI.

### Strong closing line

The YAML is intent; the proxy config is reality. I trace the handshake
from the client cluster through SDS identity to the server policy,
compare failing and healthy revisions, and restore security by rollback
rather than by turning off mTLS globally. \[16\]\[17\]\[18\]

## 10. HPA does not scale although Prometheus shows CPU above 80%

### Interview-ready answer

The first question is whether Prometheus is showing the metric that the
HPA actually consumes. A resource HPA normally reads the Kubernetes
resource metrics API, while a custom HPA reads a custom or external
metrics adapter. Prometheus CPU can be 80% of a core, 80% of a limit, or
80% of a request; those are different calculations.

I would inspect the HPA status, conditions, events, target reference,
min/max replicas, and current metric. Then I would query the same
metrics API the controller uses. I would verify CPU requests on every
relevant container because utilization is calculated relative to
requests; missing requests can make pod utilization undefined. If the
HPA requests more replicas but pods remain Pending, I continue into
scheduler, Cluster Autoscaler or Karpenter, node-group limits, EC2
quota, subnet IPs, and capacity.

### Kubernetes diagnosis

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th>kubectl describe hpa &lt;name&gt; -n &lt;ns&gt;<br />
kubectl get hpa &lt;name&gt; -n &lt;ns&gt; -o yaml<br />
kubectl top pod -n &lt;ns&gt; --containers<br />
kubectl get --raw
'/apis/metrics.k8s.io/v1beta1/namespaces/&lt;ns&gt;/pods' | jq<br />
kubectl get apiservice | grep metrics<br />
kubectl describe apiservice v1beta1.metrics.k8s.io<br />
kubectl get deploy &lt;target&gt; -n &lt;ns&gt; -o yaml<br />
kubectl get events -n &lt;ns&gt; --sort-by=.lastTimestamp</th>
</tr>
</thead>
<tbody>
</tbody>
</table>

For custom metrics:

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th>kubectl get --raw '/apis/custom.metrics.k8s.io/v1beta1' | jq<br />
kubectl get --raw '/apis/external.metrics.k8s.io/v1beta1' | jq</th>
</tr>
</thead>
<tbody>
</tbody>
</table>

### Hypothesis tree

- **Metric mismatch:** Prometheus graph uses a different query,
  aggregation, window, or denominator.

- **Missing requests:** One or more containers lack CPU requests, so
  utilization cannot be computed correctly.

- **Adapter failure:** Metrics Server or Prometheus adapter APIService
  unavailable, stale, unauthorized, or returning no series.

- **Wrong selector:** Metric labels do not match the HPA target pods.

- **HPA limits:** Already at maxReplicas, scaling disabled, or target
  reference invalid.

- **Behavior controls:** Stabilization window, tolerance, rate limit, or
  readiness delay prevents immediate scaling.

- **New pods ignored temporarily:** Initialization and readiness
  behavior can exclude metrics.

- **Scale happened, capacity did not:** Pods Pending due to CPU/memory,
  taints, affinity, PDB behavior, node-group max, EC2 quota, or subnet
  IP exhaustion.

### Cloud-level continuation

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th>kubectl get pods -n &lt;ns&gt; -o wide<br />
kubectl describe pod &lt;pending-pod&gt; -n &lt;ns&gt;<br />
kubectl get nodes -o custom-<br />
columns=NAME:.metadata.name,CPU:.status.allocatable.cpu,MEM:.status.allocatable.memory<br />
kubectl logs -n kube-system deploy/cluster-autoscaler --since=15m<br />
# or inspect Karpenter NodePool/NodeClaim events</th>
</tr>
</thead>
<tbody>
</tbody>
</table>

Check managed node group desired/max size, ASG activity, instance-type
availability, service quotas, launch-template failures, IAM, and
available pod IPs.

### Strong closing line

I never use a Prometheus screenshot to conclude the HPA is broken. I
query the exact API and formula the HPA uses, then separate replica
calculation from pod schedulability and cloud capacity. \[14\]

## 11. Sidecar-based caching layer causes tail-latency spikes

### Interview-ready answer

I would prove whether the sidecar is the cause by comparing the same
request through three paths: normal sidecar cache, cache bypass, and
direct dependency access in a controlled canary. I would split latency
into application queue time, sidecar processing, cache lookup, upstream
connection, and network time using traces and proxy stats.

Tail latency often comes from queueing rather than average service time.
I would inspect CPU throttling, event-loop saturation, memory and GC,
connection pool limits, lock contention, cache miss amplification,
synchronized TTL expiry, retries, and response-size distribution. The
first mitigation may be to lower cache concurrency, disable the cache
for a request class, stagger TTLs, increase headroom, or route a small
percentage around the sidecar.

### Evidence to collect

- p50/p95/p99/p99.9 latency by cache hit/miss, object size, region,
  node, and version.

- Cache hit ratio and miss reason, eviction rate, object count, memory,
  GC pause, and queue depth.

- Sidecar CPU usage and throttled seconds, memory working set, file
  descriptors, and sockets.

- Upstream connection-pool pending requests, resets, timeouts, and
  circuit-breaker overflow.

- Trace spans showing time before sidecar, inside sidecar, and at the
  origin.

- Node-level packet loss, retransmits, conntrack, and network queue
  drops.

### Common causes

- **Cache stampede:** Many requests miss the same key and all call the
  origin.

- **TTL synchronization:** Large key sets expire at the same boundary.

- **Single-flight bug:** A lock or request coalescer serializes
  unrelated keys.

- **CPU throttling:** Sidecar limit is too low; p50 looks acceptable
  while p99 queues.

- **Large-object copy:** Serialization or buffer copies increase with
  response size.

- **Connection pool ceiling:** Hits are fast, but misses queue behind
  too few upstream connections.

- **Retry amplification:** Cache and mesh both retry the same slow
  origin.

- **Memory pressure:** Eviction or GC cycles create periodic stalls.

- **Noisy neighbor:** Sidecar shares pod cgroup resources and competes
  with the application.

### Debug path

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th>kubectl top pod -n &lt;ns&gt; --containers<br />
kubectl describe pod &lt;pod&gt; -n &lt;ns&gt; | sed -n
'/Limits:/,/Conditions:/p'<br />
kubectl exec &lt;pod&gt; -c cache -- sh -c 'ss -s; cat
/proc/pressure/cpu; cat /proc/pressure/memory'<br />
# Envoy examples when applicable<br />
curl -s localhost:9901/stats | grep -E
'pending|overflow|timeout|reset|rq_time'</th>
</tr>
</thead>
<tbody>
</tbody>
</table>

### Durable fixes

- Randomize TTLs and use stale-while-revalidate.

- Coalesce only identical keys with bounded waiters.

- Set independent retry budgets and deadlines.

- Reserve CPU and memory based on p99 load, not average load.

- Separate cache from application lifecycle if resource isolation is
  essential.

- Add cache-hit/miss labels to traces and SLO dashboards.

- Load test realistic key popularity and object-size distributions.

### Strong closing line

A cache improves average latency but can worsen tail latency if misses,
expiration, or queueing synchronize. I isolate the sidecar with a bypass
experiment, decompose the latency, and fix the queueing mechanism rather
than simply adding replicas.

## 12. Playback 504s while cloud LB and sidecars look healthy

### Interview-ready answer

A healthy load balancer and healthy sidecars only prove that shallow
checks pass. I would run an end-to-end synthetic that follows the real
playback path: edge request, authentication, manifest generation,
entitlement or DRM, origin selection, and actual segment byte-range
retrieval. Then I trace one failed request using a request ID across the
CDN/edge, gateway, mesh, playback service, metadata, origin, and object
storage.

A 504 identifies the component that timed out waiting for its upstream;
it does not identify the root cause. I would inspect timeout hierarchy,
upstream response flags, connection pools, DNS, endpoint health,
retries, large-response streaming, range requests, and dependency
saturation. The immediate mitigation is to route around the unhealthy
pool or dependency, reduce retry amplification, and degrade nonessential
features while preserving core playback.

### Layer-by-layer path

1\. Client and ISP path.

2\. CDN or global edge.

3\. Cloud load balancer and gateway.

4\. Mesh route and sidecar.

5\. Playback API or manifest service.

6\. Identity, entitlement, DRM/license, metadata, recommendation, and
personalization dependencies.

7\. Origin selection, object storage, cache, or transcoded segment
store.

8\. Response streaming back through every timeout boundary.

### What to inspect

- Gateway access log: route, upstream cluster, response flags, duration,
  bytes sent.

- Envoy stats: upstream_rq_timeout, upstream_cx_connect_timeout, pending
  overflow, resets, no healthy upstream.

- Distributed trace: which span consumes the deadline?

- Dependency saturation: queue depth, thread pool, connection pool,
  database locks, object-store latency.

- Request class: title, region, device, codec, bitrate, manifest type,
  object size, or byte range.

- Timeout order: client \> edge \> gateway \> service \> dependency,
  with each inner deadline shorter than the outer one.

### Important hidden failures

- Health endpoint is local while the business dependency is unavailable.

- A route sends only large or long-lived requests to an
  under-provisioned pool.

- Connection-pool exhaustion creates queue time with low CPU.

- DNS resolves to a reachable but wrong or stale endpoint.

- MTU, packet loss, or retransmission affects large segment transfers
  but not tiny health checks.

- Retry policies multiply requests across gateway, sidecar, and client.

- Object storage or CDN rejects byte-range behavior.

- A downstream service returns slowly and the gateway converts the
  deadline into 504.

### Mitigation

- Shift traffic by region, availability zone, endpoint subset, or
  service version.

- Serve a less personalized or cached manifest.

- Disable a noncritical dependency and use defaults.

- Reduce or disable retries when they worsen overload.

- Increase capacity only after confirming the bottleneck; blindly
  scaling the caller can crush the dependency.

### Strong closing line

I define health from the user's ability to start and continue streaming,
not from HTTP 200 on infrastructure probes. I trace a real segment
request end to end, find the first exhausted deadline, and route around
or degrade the smallest failing dependency.

## 13. NAT costs surge with no infrastructure change

### Interview-ready answer

I would treat NAT spend as a traffic-forensics incident. NAT charges
increase with bytes processed, so "no infrastructure change" does not
mean no traffic-path change. I would identify the NAT gateway, region,
usage type, and time window in billing data, then correlate CloudWatch
NAT byte and connection metrics with VPC Flow Logs to find top source
ENIs, destinations, ports, and availability zones.

Common silent triggers are a private endpoint or private DNS failure
that sends cloud-service traffic to public IPs, container image or
package downloads, telemetry export changes, retries after a dependency
slowdown, cross-AZ routing through a NAT, backup or reindex jobs,
malware or credential abuse, and applications resolving a service to its
public endpoint.

### Investigation sequence

1\. Cost Explorer or CUR: region, account, NAT gateway, processing
bytes, and transfer categories.

2\. CloudWatch: bytes in/out, active connections, connection attempts,
port allocation errors, packet drops.

3\. Flow Logs: aggregate by source ENI, destination address, destination
port, bytes, and AZ.

4\. Map source ENIs to pods, nodes, Lambda functions, databases, or
endpoints.

5\. Resolve top destinations and compare with route-table and VPC
endpoint expectations.

6\. Check deployments, image tags, telemetry agents, backup jobs, DNS
changes, and application retry rates.

### Silent causes to name in the interview

- S3, ECR, STS, CloudWatch, or other cloud API traffic stopped using a
  VPC endpoint.

- Private DNS was disabled or a custom resolver returned public
  addresses.

- A new base image downloads packages at startup.

- Pods pull uncached images repeatedly after node churn.

- Cross-AZ traffic reaches a NAT in another zone.

- A logging exporter changed from regional/private to internet endpoint.

- Retry storms increase bytes without a deployment.

- Data replication, backup, or model download volume changed.

- Compromised workload exfiltrates or scans externally.

### Fixes

- Restore gateway or interface endpoints and private DNS.

- Use a NAT gateway per AZ where architecture and economics justify it,
  with same-AZ routing.

- Cache images and packages; use private artifact repositories.

- Add egress policy and destination allowlists.

- Create cost anomaly alerts correlated with NAT byte metrics.

- Tag and attribute egress by account, VPC, subnet, workload, and
  destination.

### Strong closing line

NAT cost is a symptom of an unexpected data path. I find the top talker
and destination with billing, NAT metrics, and flow logs, then fix
routing or application behavior rather than merely setting a budget
alert. \[19\]

# Round 3 - Leadership, Chaos Culture, and Engineering Influence

## 14. Building a culture where SLOs are owned, not ignored

### Interview-ready answer

I make SLOs a product decision with engineering enforcement, not an SRE
dashboard. Each critical service has a small number of user-centered
SLIs, a named product and engineering owner, a documented measurement
window, and an agreed error-budget policy. The policy changes behavior:
when the budget is healthy, teams can release; when burn is excessive,
reliability work and release risk are prioritized.

I would start with a few customer journeys, not hundreds of component
metrics. Teams review burn rate in the same forum where they review
delivery. Alerts page on fast and slow error-budget burn, not every CPU
spike. Reliability work enters planning with the same visibility as
features. Postmortems assign systemic actions with owners and due dates,
and leadership evaluates whether teams improve outcomes rather than
whether they avoided all incidents.

### Adoption plan

1\. Pick 3-5 tier-1 user journeys.

2\. Define availability, latency, freshness, and correctness SLIs at the
user-observable boundary.

3\. Agree on SLO targets and exclusions with product, engineering, and
operations.

4\. Publish an error-budget policy with release and escalation
consequences.

5\. Build shared dashboards and multi-window burn-rate alerts.

6\. Add SLO review to service launch, quarterly planning, and incident
review.

7\. Track action completion and recurring budget consumers.

### Ownership model

- **Product:** Defines user impact and acceptable risk.

- **Service team:** Owns implementation and operational response.

- **Platform/SRE:** Provides measurement, tooling, coaching, and
governance.

- **Leadership:** Enforces the policy when feature pressure conflicts
  with reliability evidence.

### Anti-patterns

- 100% SLOs that make every error a crisis.

- SLOs based on what is easy to measure rather than what users
  experience.

- Platform team owns every service's reliability while product teams own
  only features.

- Error budgets are used as punishment.

- Dashboards exist, but no planning or release decision changes.

- Teams exclude incidents until the number looks good.

### Metrics of cultural success

- Percentage of tier-1 services with approved SLO and owner.

- Percentage of incidents tied to an SLI and budget impact.

- Repeat-incident rate and action-item completion.

- Change failure rate, recovery time, and customer-impact minutes.

- Reliability work planned before budget exhaustion rather than after a
  crisis.

### Strong closing line

An SLO becomes real when it changes a release or investment decision. I
build shared ownership by making the error budget the neutral contract
between product velocity and customer reliability. \[20\]\[21\]

## 15. Multi-region failover in three weeks with no DNS layer

### Interview-ready answer

In three weeks I would constrain scope and choose active-passive unless
the state model is already active-active. "No DNS" does not mean no
traffic steering; it means I need a stable anycast or global front-door
address, such as a global accelerator, CDN, or provider-neutral anycast
service that can health-check regional endpoints and move new
connections without waiting for DNS TTLs.

The hard part is not the front door. It is state, write ownership,
dependency readiness, capacity, and failback. I would define RTO/RPO,
inventory stateful dependencies, replicate required data, pre-provision
secondary capacity, create health checks based on a real transaction,
implement write fencing, and rehearse failover and failback. If those
cannot be proven in three weeks, I would be explicit that we can deliver
traffic failover but not safe full-service failover.

### Three-week plan

\#### Week 1 - scope and foundation

- Define critical user journey, RTO, RPO, and acceptable degraded mode.

- Choose secondary region and verify quotas, images, networking,
  certificates, secrets, and dependencies.

- Deploy infrastructure from the same immutable IaC and application
  artifacts.

- Establish data replication and identify non-replicated state.

- Put both regional endpoints behind a stable global address.

\#### Week 2 - correctness and observability

- Create deep regional health checks and synthetic transactions.

- Implement traffic weights and one-command rollback.

- Add write fencing or leader election to prevent split brain.

- Validate secret/certificate rotation, queues, object storage,
  databases, and third-party allowlists.

- Load test the secondary at expected failover capacity.

\#### Week 3 - game days and launch

- Fail a single dependency, AZ, regional endpoint, and primary region.

- Measure detection, decision, traffic shift, error rate, and recovery.

- Test failback after data reconciliation.

- Freeze unrelated changes, document runbooks, assign incident roles,
  and gain business sign-off on residual risks.

### Architecture choices

- Stable anycast IPs or a global traffic service direct new connections
  to healthy regional load balancers.

- Existing long-lived connections may require client reconnect or
  graceful drain.

- Session state is externalized or clients can re-authenticate.

- Writes are single-region unless conflict resolution and consistency
  have already been designed.

- Dependencies are local or have explicit cross-region failure behavior.

### Strong closing line

I would not promise "multi-region" as a routing feature. In three weeks
I can deliver a carefully scoped, active-passive recovery path with a
stable global address, but I will prove state integrity, capacity, write
fencing, and failback before calling it production-ready. \[22\]

## 16. Chaos and graceful-degradation testing for a major release

### Interview-ready answer

I start with a steady-state hypothesis tied to customer SLIs: playback
start success, rebuffer rate, manifest latency, and segment
availability. Then I inject one controlled fault at a time with a small
blast radius, automated stop conditions, and an incident owner. The
purpose is not to prove the system can fail; it is to prove the user
journey remains within an agreed degraded envelope and that the
organization can detect, decide, and recover.

For a major release, I would combine production-like load tests, dark or
shadow traffic, canary release, dependency fault injection, AZ and node
failures, network latency and packet loss, DNS and certificate failures,
cache loss, queue backlog, and controlled regional evacuation. I would
also test graceful degradation: disable personalization, serve cached
metadata, lower bitrate, use stale-but-safe data, or reduce nonessential
writes while preserving playback.

### Experiment template

- **Hypothesis:** If one AZ loses 40% of playback capacity,
  start-success remains above X and p99 manifest latency below Y.

- **Blast radius:** One canary cluster, 1% traffic, one title class, or
  internal users.

- **Fault:** Terminate nodes, introduce latency, block a dependency,
  expire a test certificate, or throttle storage.

- **Observability:** User SLI, error-budget burn, queue depth, retries,
  saturation, and regional capacity.

- **Stop conditions:** CloudWatch/Prometheus alarm, error rate, latency,
  or customer-impact threshold.

- **Recovery:** Automatic rollback, traffic shift, capacity restore, and
  data validation.

- **Learning:** Detection gap, unsafe retry, missing fallback, runbook
  defect, or capacity assumption.

### Release-specific scenarios

- Edge or gateway config canary fails.

- One control plane is unavailable while proxies continue with
  last-known config.

- Cache cluster is cold or partially unavailable.

- Metadata or personalization service is slow; playback uses defaults.

- DRM/license dependency has high latency.

- Object storage returns throttling or elevated tail latency.

- DNS returns stale or empty data.

- Certificate rotation overlaps the release.

- One region loses network or write availability.

- Autoscaling is delayed while traffic rises sharply.

### Safety rules

- Begin in staging, but do not stop there; staging rarely reproduces
  production traffic and dependency behavior.

- Use progressive blast radius and explicit abort alarms.

- Do not run an experiment when observability or rollback is impaired.

- Protect stateful systems from destructive experiments unless restore
  is proven.

- Announce experiments and distinguish them from real incidents while
  preserving realistic on-call response.

### Strong closing line

Chaos engineering is a controlled test of a business hypothesis. The win
is not that components survive; the win is that customer impact stays
bounded, fallbacks activate, operators detect the right signal, and
recovery is repeatable. \[23\]\[24\]

## 17. Proving the ROI of infrastructure modernization to nontechnical executives

### Interview-ready answer

I translate modernization into revenue protection, delivery speed,
operating cost, and risk. I establish a baseline before the program:
deployment lead time, deployment frequency, change failure rate,
recovery time, incident customer-minutes, cloud cost per transaction or
stream, platform toil, and security/compliance effort. Then I connect
each modernization investment to a measurable business mechanism.

For example, a standardized platform may reduce environment setup from
weeks to hours, increase safe releases, lower recovery time, remove
duplicated tooling, and reduce cloud waste. I would present a one-page
scorecard with baseline, target, current result, financial range,
assumptions, and confidence. I avoid claiming every avoided outage as
guaranteed savings; I separate realized savings, capacity released, and
risk-adjusted avoidance.

### ROI model

**Annual benefit =**

- Realized infrastructure savings

- Plus engineer hours returned to product work

- Plus reduced incident and support cost

- Plus revenue protected or enabled

- Plus compliance/audit effort avoided

- Minus ongoing platform operating cost

**Investment =** migration labor + tooling + training + parallel-run
cost + organizational change.

Report payback period, three-year NPV where appropriate, and a range
rather than a fake precise number.

### Example executive scorecard

- Environment provisioning: 10 business days -\> 45 minutes.

- Deployment frequency: weekly -\> daily for qualified services.

- Change failure rate: 18% -\> 7%.

- Median recovery time: 95 minutes -\> 28 minutes.

- Cloud cost per 1,000 transactions: down 14%.

- Platform support tickets: down 35%.

- Tier-1 services with SLOs and automated rollback: 20% -\> 85%.

### Making the case credible

- Tie results to a pilot group and compare before/after or control
  groups.

- Show adoption, not just platform availability.

- Include migration and operating costs.

- Separate hard-dollar savings from capacity and risk reduction.

- Use customer-impact minutes and revenue windows for reliability value.

- Explain the counterfactual: what happens if the company does nothing?

### Executive narrative

"We are not buying Kubernetes or rewriting CI. We are reducing the time
and risk of turning an idea into a reliable customer capability. The
program pays back through fewer duplicated environments, faster safe
releases, shorter incidents, and lower unit infrastructure cost. Here is
the baseline, the pilot evidence, the three-year range, and the stop/go
checkpoints."

### Strong closing line

Executives do not need a tour of our tools. They need evidence that
modernization changes unit economics, delivery speed, and operational
risk. I make the investment measurable before it starts and create
checkpoints where leadership can expand, redirect, or stop it.

# Rapid-Fire Command and Evidence Cheat Sheet

## Envoy and Istio

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th>istioctl proxy-status<br />
istioctl analyze -A<br />
istioctl proxy-config listeners &lt;pod&gt; -n &lt;ns&gt;<br />
istioctl proxy-config routes &lt;pod&gt; -n &lt;ns&gt;<br />
istioctl proxy-config clusters &lt;pod&gt; -n &lt;ns&gt;<br />
istioctl proxy-config endpoints &lt;pod&gt; -n &lt;ns&gt;<br />
istioctl proxy-config secret &lt;pod&gt; -n &lt;ns&gt;<br />
<br />
curl -s localhost:9901/config_dump &gt; config.json<br />
curl -s localhost:9901/clusters<br />
curl -s
'localhost:9901/stats?filter=upstream.*(timeout|reset|overflow)'</th>
</tr>
</thead>
<tbody>
</tbody>
</table>

## Kubernetes health and autoscaling

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th>kubectl describe node &lt;node&gt;<br />
kubectl get events -A --sort-by=.lastTimestamp<br />
kubectl describe hpa &lt;hpa&gt; -n &lt;ns&gt;<br />
kubectl get --raw
'/apis/metrics.k8s.io/v1beta1/namespaces/&lt;ns&gt;/pods'<br />
kubectl top pod -n &lt;ns&gt; --containers<br />
kubectl describe pod &lt;pod&gt; -n &lt;ns&gt;<br />
kubectl auth can-i --list
--as=system:serviceaccount:&lt;ns&gt;:&lt;sa&gt;</th>
</tr>
</thead>
<tbody>
</tbody>
</table>

## Linux node evidence

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th>systemctl --failed<br />
systemctl show &lt;unit&gt; -p ActiveState -p SubState -p Result -p
NRestarts<br />
journalctl -u &lt;unit&gt; --since '-30 min' --no-pager<br />
dmesg -T | tail -200<br />
ss -s<br />
cat /proc/pressure/{cpu,memory,io}<br />
cat /proc/sys/net/netfilter/nf_conntrack_count<br />
cat /proc/sys/net/netfilter/nf_conntrack_max</th>
</tr>
</thead>
<tbody>
</tbody>
</table>

## Terraform recovery

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th>terraform state pull &gt; state-backup.json<br />
terraform plan -refresh-only -out=refresh.tfplan<br />
terraform show refresh.tfplan<br />
terraform force-unlock &lt;LOCK_ID&gt; # only after proving stale
ownership</th>
</tr>
</thead>
<tbody>
</tbody>
</table>

## Cilium and Hubble

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th>cilium status --verbose<br />
cilium connectivity test<br />
hubble observe --verdict DROPPED --follow<br />
hubble observe --from-pod &lt;ns&gt;/&lt;pod&gt; --follow<br />
cilium monitor --type drop<br />
cilium bpf policy get</th>
</tr>
</thead>
<tbody>
</tbody>
</table>

# Interviewer Follow-Up Traps

## "Would you just restart it?"

A strong answer: "A restart may be a mitigation after evidence capture,
but first I need to know whether restart is the cure or merely erases
the evidence. For a repeated node-level failure I cordon, drain, and
replace from a known-good image."

## "Why not increase the timeout?"

A strong answer: "Increasing an outer timeout can hide queueing and
consume more concurrency. I first identify which dependency uses the
deadline and why. Any timeout change must preserve a coherent hierarchy
and capacity model."

## "Why not disable mTLS temporarily?"

A strong answer: "My first mitigation is rollback or traffic shift.
Disabling mTLS expands the security blast radius and may break
identity-based authorization. I would only use a narrowly scoped,
time-limited exception with security approval."

## "Why not run Terraform with -lock=false?"

A strong answer: "Because the outage has already made state authority
uncertain. Allowing another writer creates the exact corruption scenario
locking is intended to prevent."

## "Prometheus says 80%; why are you checking Metrics Server?"

A strong answer: "Because the HPA acts on the metrics API configured in
its spec. A dashboard query can use a different denominator,
aggregation, or time window. I must inspect the controller's
authoritative input."

# Principal-Level Language That Improves Answers

Use these phrases naturally:

- "I would separate the control plane from the data plane and define the
  stale-config behavior."

- "I need to identify the authoritative signal for this controller."

- "I would bound the failure by region, revision, request class, and
  dependency."

- "The immediate mitigation and the durable correction are different
  decisions."

- "I would preserve evidence before restarting or replacing the
  component."

- "The rollback criterion is an SLI, not whether the deployment command
  succeeded."

- "I would test the failure mode with a progressive blast radius and an
  automated stop condition."

- "I am optimizing for independent survivability, not theoretical
  portability."

- "I would explicitly document the residual risk and the point at which
  the business must choose scope, time, or reliability."

# Mock Interview Drills

## Drill 1 - 90-second architecture answer

Pick questions 1, 2, 3, or 15. Record yourself. Score 0-2 on each:

- Assumptions and objective stated.

- Clear layered design.

- Failure behavior explained.

- Operability and rollout included.

- Trade-off acknowledged.

Target: 8/10 without exceeding 100 seconds.

## Drill 2 - Five-minute incident answer

Pick questions 9-13. Use STABILIZE. You must include:

- Immediate blast-radius control.

- One failing-request trace.

- Three competing hypotheses.

- Authoritative commands or data sources.

- Safest mitigation.

- Prevention and measurable rollout guardrail.

## Drill 3 - Whiteboard challenge

Draw four boxes: user edge, regional gateway, service mesh, dependency.
Add control plane above and telemetry below. Explain:

- What happens when the control plane is down?

- Where identity is established?

- Where timeouts and retries live?

- Which health check represents the user?

- How traffic shifts during failure?

## Drill 4 - Leadership answer

For questions 14 or 17, avoid tool names for the first 60 seconds. Speak
in customer impact, release risk, unit cost, ownership, and decision
policy.

# Seven-Day Preparation Plan

## Day 1 - Service mesh and discovery

- Master questions 1, 7, and 9.

- Practice istioctl proxy-status and all proxy-config views.

- Explain xDS ACK/NACK and last-known-good behavior from memory.

## Day 2 - eBPF, networking, and NAT

- Master questions 2 and 13.

- Write one default-deny Cilium policy and debug a denied flow.

- Practice mapping Flow Log ENIs to workload owners.

## Day 3 - EKS nodes and custom images

- Master questions 4 and 5.

- Build a node-failure decision tree: restart, cordon, drain, replace.

- Describe your AMI promotion and rollback process in under two minutes.

## Day 4 - Probes and autoscaling

- Master questions 6 and 10.

- Write separate startup, readiness, and liveness contracts.

- Explain CPU utilization versus raw CPU usage.

## Day 5 - User-path RCA

- Master questions 11 and 12.

- Practice tracing one request through gateway, sidecar, service, and
  dependency.

- Explain why healthy infrastructure probes can coexist with user
  failure.

## Day 6 - Multi-cloud, multi-region, and chaos

- Master questions 3, 15, and 16.

- Define RTO/RPO, write fencing, failover, and failback.

- Create one chaos hypothesis with stop conditions.

## Day 7 - SLO leadership and ROI

- Master questions 14 and 17.

- Prepare a two-minute SLO adoption story and a two-minute ROI story.

- Run a full 60-minute mock interview and review filler words,
  structure, and missing trade-offs.

# Personal Story Mapping Worksheet

For each category, prepare one real story using Situation, Constraint,
Decision, Evidence, Result, and Learning.

- Service discovery, ingress, or networking failure.

- Kubernetes node or runtime incident.

- Terraform state or infrastructure change risk.

- Autoscaling or capacity failure.

- Security or identity rollout.

- Major incident with a difficult mitigation decision.

- Platform standardization or modernization with measurable outcome.

- Leadership conflict where reliability competed with delivery pressure.

The best answer combines the scenario design in this pack with one
concrete production story. Your experience is the differentiator; the
product vocabulary is only the translation layer.

# Official Reference Guide

The following primary sources were used to verify technical behavior.
Product features and commands should still be checked against the exact
versions used by the interviewing organization.

| **Official source**                                                | **Link**                                                                                                                            |
|--------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| \[1\] Envoy - Service discovery                                    | [<u>Open official documentation</u>](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/service_discovery)    |
| \[2\] Envoy - xDS REST and gRPC protocol                           | [<u>Open official documentation</u>](https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol)                             |
| \[3\] Istio - Architecture                                         | [<u>Open official documentation</u>](https://istio.io/latest/docs/ops/deployment/architecture/)                                     |
| \[4\] Istio - Traffic management                                   | [<u>Open official documentation</u>](https://istio.io/latest/docs/concepts/traffic-management/)                                     |
| \[5\] Istio - DNS proxying and DNS behavior                        | [<u>Open official documentation</u>](https://istio.io/latest/docs/ops/configuration/traffic-management/dns-proxy/)                  |
| \[6\] Cilium and Hubble - Introduction                             | [<u>Open official documentation</u>](https://docs.cilium.io/en/stable/overview/intro/)                                              |
| \[7\] Cilium - eBPF datapath introduction                          | [<u>Open official documentation</u>](https://docs.cilium.io/en/stable/network/ebpf/intro/)                                          |
| \[8\] Cilium - Hubble internals and flow visibility                | [<u>Open official documentation</u>](https://docs.cilium.io/en/stable/internals/hubble/)                                            |
| \[9\] Amazon EKS - Node health monitoring and automatic repair     | [<u>Open official documentation</u>](https://docs.aws.amazon.com/eks/latest/userguide/node-health.html)                             |
| \[10\] Kubernetes - Monitor node health / Node Problem Detector    | [<u>Open official documentation</u>](https://kubernetes.io/docs/tasks/debug/debug-cluster/monitor-node-health/)                     |
| \[11\] EC2 Image Builder - Components, pipelines, and tests        | [<u>Open official documentation</u>](https://docs.aws.amazon.com/imagebuilder/latest/userguide/manage-components.html)              |
| \[12\] Amazon Inspector - EC2 scanning                             | [<u>Open official documentation</u>](https://docs.aws.amazon.com/inspector/latest/user/scanning-ec2.html)                           |
| \[13\] Kubernetes - Liveness, readiness, and startup probes        | [<u>Open official documentation</u>](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/) |
| \[14\] Kubernetes - Horizontal Pod Autoscaling                     | [<u>Open official documentation</u>](https://kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/)           |
| \[15\] Terraform - State locking and backend recovery behavior     | [<u>Open official documentation</u>](https://developer.hashicorp.com/terraform/language/state/locking)                              |
| \[16\] Istio - Understanding TLS configuration                     | [<u>Open official documentation</u>](https://istio.io/latest/docs/ops/configuration/traffic-management/tls-configuration/)          |
| \[17\] Istio - Debugging Envoy and Istiod                          | [<u>Open official documentation</u>](https://istio.io/latest/docs/ops/diagnostic-tools/proxy-cmd/)                                  |
| \[18\] Envoy - Admin interface, config dump, and stats             | [<u>Open official documentation</u>](https://www.envoyproxy.io/docs/envoy/latest/start/quick-start/admin)                           |
| \[19\] AWS VPC - NAT gateway metrics and VPC Flow Logs             | [<u>Open official documentation</u>](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway-cloudwatch.html)              |
| \[20\] Google SRE - Service Level Objectives                       | [<u>Open official documentation</u>](https://sre.google/sre-book/service-level-objectives/)                                         |
| \[21\] Google SRE Workbook - Example error budget policy           | [<u>Open official documentation</u>](https://sre.google/workbook/error-budget-policy/)                                              |
| \[22\] AWS Global Accelerator - How health-based failover works    | [<u>Open official documentation</u>](https://docs.aws.amazon.com/global-accelerator/latest/dg/introduction-how-it-works.html)       |
| \[23\] AWS Fault Injection Service - Chaos engineering experiments | [<u>Open official documentation</u>](https://docs.aws.amazon.com/fis/latest/userguide/what-is.html)                                 |
| \[24\] Kubernetes - Disruptions and PodDisruptionBudgets           | [<u>Open official documentation</u>](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)                               |
| \[25\] AWS / Azure / Google - Workload identity federation         | [<u>Open official documentation</u>](https://docs.aws.amazon.com/eks/latest/userguide/service-accounts.html)                        |
| \[26\] HashiCorp Vault - Secrets sync and Kubernetes integration   | [<u>Open official documentation</u>](https://developer.hashicorp.com/vault/docs/sync)                                               |

# Final Interview Reminder

| Do not answer like a runbook reader. Answer like the person accountable for customer impact, security, recovery, and the next incident. State assumptions, use authoritative evidence, choose the safest mitigation, and make the trade-off explicit. |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

**You have the experience. This pack gives it a sharp
platform-engineering vocabulary and a disciplined interview structure.**
