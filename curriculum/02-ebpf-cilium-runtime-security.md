# Netflix Adapter — eBPF Runtime Security for a Large Streaming Platform

> **Original scenario:** Design an eBPF-based runtime network-security architecture for a large Kubernetes platform. Explain where Cilium enforces policy, how Hubble provides evidence, how runtime detection differs from inline prevention, how Falco and Tetragon fit, what fails under extreme load, and how you migrate safely without creating a node-wide security or availability incident.

This is a thin Netflix/media-platform adapter. Reusable Linux, eBPF, Cilium, Hubble, Falco, Tetragon, policy, detection, enforcement, migration, and failure-mode theory is canonical in the shared handbook.

## Canonical prerequisites

- [eBPF, Cilium, Hubble, Falco, and Tetragon runtime security](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/core/ebpf-security/cilium-hubble-falco-tetragon.md)
- [Linux networking, containers, cgroups, and security](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/core/linux/05-networking-containers-security.md)
- [Linux observability and production debugging](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/core/linux/06-observability-debugging.md)
- [Kubernetes Service, DNS, Ingress, and Gateway request paths](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/core/kubernetes/networking/service-dns-ingress-gateway-request-path.md)
- [Kubernetes node-image qualification, promotion, and rollback](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/core/kubernetes/node-images/qualification-promotion-rollback.md)
- [Chaos engineering and game-day governance](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/core/reliability/chaos-engineering-game-days.md)

## Netflix-shaped concerns

The scenario adds:

- very high east-west connection and request volume;
- playback-critical paths with strict tail-latency budgets;
- many clusters, cells, regions, kernel versions, and node-image cohorts;
- rapid application and policy change;
- high-cardinality flow evidence;
- optional analytics and enrichment paths that must not consume security or dataplane capacity needed by playback;
- a requirement that a bad BPF program, agent, policy, or image remain contained.

## Ninety-second answer

> I would separate four responsibilities. Cilium provides the Kubernetes dataplane and identity-aware network policy. Hubble provides flow evidence from that dataplane. Falco provides broad runtime detection across syscall and audit sources. Tetragon provides eBPF-native process, file, network, and identity-aware detection and selected inline enforcement where the policy is precise enough to fail safely.
>
> I would not begin by replacing the whole fleet. First I would inventory kernels, cgroups, CNIs, kube-proxy modes, host-network workloads, DNS, Service, ingress, egress, and observability dependencies. I would qualify a pinned node image, run compatibility and failure tests, deploy to a canary node pool, observe duplicate policy in audit mode, then migrate by cell and failure domain with explicit rollback to the prior image and dataplane.
>
> Playback-critical traffic gets protected dataplane and telemetry capacity. Policy distribution is versioned and staged. Expensive runtime rules are measured and bounded. Hubble and security events are sampled or prioritized without hiding loss. Enforcement begins with high-confidence invariants such as forbidden executable paths, unexpected privilege, or disallowed network identity—not broad behavioral guesses that can kill healthy workloads.
>
> I would stop rollout on packet loss, latency, agent crash, map pressure, verifier rejection, policy-convergence failure, or security-visibility loss. Recovery is proven through playback and service SLIs, dataplane health, policy convergence, flow evidence, and successful rollback—not merely that the Cilium pods are Running.

## Responsibility model

| Layer | Primary purpose | Netflix-specific interpretation |
|---|---|---|
| Cilium | pod networking, Service translation, network policy, encryption where configured | protect playback and control traffic without global policy fan-out or dataplane collapse |
| Hubble | flow visibility and policy verdict evidence | compare healthy and affected cells, versions, identities, endpoints, and paths |
| Falco | runtime detection from syscall and other sources | broad detection coverage, including suspicious host/container behavior |
| Tetragon | eBPF process/network/file observability and selective enforcement | high-confidence workload and host invariants with bounded inline action |

## Architecture

```text
Kubernetes identity and policy intent
          |
          v
Cilium agents and operator
  - endpoint identity
  - Service/dataplane state
  - policy maps
  - encryption and routing where enabled
          |
          +------> Hubble flow evidence
          |
          +------> Tetragon process/network/file evidence and selected enforcement
          |
          +------> Falco runtime detections
          |
          v
regional security pipeline
  - normalize
  - prioritize
  - correlate by cell, service, image, kernel, and release
  - route urgent action versus investigation
```

## Policy strategy

1. Establish identity and visibility before broad enforcement.
2. Start with default-deny only in bounded namespaces or service classes with tested dependencies.
3. Keep DNS, control-plane, node, storage, observability, and emergency-access paths explicit.
4. Use application/service identity rather than mutable IP addresses.
5. Separate network policy from runtime detection and host enforcement.
6. Stage policy in audit or visibility mode where supported.
7. Roll out by node-image, cluster, cell, service class, and region.
8. Retain one-command or one-version rollback to known-good state.

## Playback-critical failure model

### Dataplane agent failure

- determine whether existing BPF state continues serving;
- detect stale policy, endpoint, Service, and encryption state;
- stop node-image or agent rollout;
- cordon or replace only affected nodes when necessary;
- avoid a synchronized agent restart across the fleet.

### Policy error

- compare accepted policy version and affected identity/path;
- revert the specific policy or rollout ring;
- preserve flow verdicts and control-plane evidence;
- do not disable security globally as the first response.

### Kernel or BPF incompatibility

- isolate by kernel, image, instance type, and architecture;
- preserve verifier, kernel, agent, and crash evidence;
- stop promotion and restore known-good nodes;
- qualify the correction through the immutable image pipeline.

### Telemetry overload

- preserve policy-deny, security, and playback-critical evidence;
- shed low-value debug flows first;
- expose dropped-event and sampling metrics;
- enforce tenant, cluster, and signal quotas;
- prevent the security pipeline from consuming application capacity.

## Migration plan

```text
inventory and compatibility matrix
  -> disposable cluster
  -> representative workload tests
  -> canary node pool
  -> shadow/audit policy
  -> one low-risk production cell
  -> playback-critical synthetic validation
  -> regional rollout rings
  -> old dataplane retirement
```

Promotion gates include:

- node bootstrap and Cilium readiness;
- pod sandbox and Service connectivity;
- DNS and NetworkPolicy correctness;
- packet loss, resets, and tail latency;
- BPF map and memory pressure;
- policy convergence;
- Hubble/security event completeness;
- kernel warnings and agent restarts;
- playback-start, manifest, entitlement, DRM, and segment SLIs;
- tested rollback.

## Evidence and SLOs

Track:

- endpoint and policy convergence time;
- dropped packets and policy denies by identity/path;
- BPF map occupancy and allocation failure;
- agent/operator restart and error rate;
- verifier rejection and kernel warning rate;
- encryption and tunnel health where used;
- Hubble/security event acceptance, drop, and freshness;
- CPU/memory overhead by node image and workload class;
- Service and DNS success;
- playback-start latency, rebuffer rate, and fatal playback errors by cell and dataplane cohort.

## Adversarial follow-ups

### Why not enforce every suspicious behavior inline?

Detection rules often have uncertainty and changing application behavior. Inline enforcement is reserved for precise invariants whose false-positive and failure behavior are understood and tested.

### What if the Cilium agent dies?

Existing kernel state may continue forwarding for a period, but endpoint, policy, identity, and Service state can become stale. Measure the real last-known-good window and replace or repair a bounded node cohort.

### What if security telemetry is dropped during peak load?

Loss must be explicit. Prioritize critical verdicts and high-confidence events, enforce quotas, shed lower-value debug data, and maintain a separate durability contract for audit events that cannot be lost.

### Why is a node-image rollback part of security design?

Kernel, CNI, BPF, runtime, and security-agent compatibility are coupled. A safe migration needs an immutable known-good image and controlled node replacement, not live host mutation.

### How do you avoid a fleet-wide security outage?

Canary node pools, cell boundaries, bounded concurrency, stop thresholds, preserved old capacity, policy staging, systemic-failure circuit breakers, and tested rollback.

## Whiteboard flow

```text
1. classify dataplane, flow evidence, detection, and enforcement roles
2. inventory kernel/CNI/workload compatibility
3. pin and qualify node image plus agent versions
4. observe identities and flows before broad enforcement
5. stage policy and runtime rules by risk
6. canary by node pool and cell
7. stop on latency, loss, crash, map, convergence, or evidence regression
8. recover through known-good images and policies
9. prove playback and security SLIs
```

## Personal-story bridge

Use a truthful story about Linux/kernel troubleshooting, Kubernetes networking, security controls, immutable infrastructure, high-volume monitoring, staged migration, or stopping automation after detecting a systemic pattern. Keep production evidence separate from hypothetical streaming scale.
