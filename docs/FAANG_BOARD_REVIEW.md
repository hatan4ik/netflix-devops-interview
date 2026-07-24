# FAANG Engineering Board Review - Principal / Staff Calibration

> Independent technical review of the Netflix-scale DevOps and Platform Engineering interview pack.
>
> This material describes hypothetical interview scenarios. It does not claim to represent Netflix's internal architecture or official interview process.

## Executive verdict

| Material | Board score | Hiring signal |
|---|---:|---|
| Uncorrected briefing | 6.2/10 | Strong senior instincts, but several category errors create a Staff/Principal no-hire risk |
| Corrected training pack | 8.9/10 | Strong Senior / Staff performance |
| Board-refined delivery | 9.3+/10 | Competitive Principal-level performance |

The core technical content is strong. The top-1% improvement is not adding more product names. It is consistently demonstrating:

- explicit requirements and invariants;
- failure-domain and blast-radius reasoning;
- separation of control plane, data plane, and application behavior;
- evidence-driven diagnosis;
- safe mitigation with rollback criteria;
- measurable validation;
- prevention that reduces recurrence probability and impact.

## Hiring committee summary

### Uncorrected material, delivered verbatim

**No-hire for Principal.**

The candidate sounds experienced but overconfident. The damaging statements are the ones that collapse distinct systems into a single explanation, such as treating Cilium network policy as syscall enforcement, using the Descheduler as a node-repair controller, making shared dependency failures trigger liveness restarts, assuming EDS makes DNS irrelevant, or treating Terraform force-unlock as a routine action.

### Corrected material, delivered competently

**Strong hire at Senior; likely hire at Staff.**

It demonstrates production judgment, security awareness, state integrity, Kubernetes control-loop understanding, measured incident response, and strong leadership instincts.

### Corrected material plus this board calibration

**Competitive Principal-level performance.**

The expected voice is:

> I optimize for restoring the customer journey without corrupting state, weakening security, or expanding the failure domain. I make the failure model explicit, change one layer at a time, define measurable acceptance criteria, and ensure the permanent fix reduces both recurrence probability and blast radius.

---

# Board scorecard

| Scenario | Initial risk | Board-refined result | Final emphasis |
|---|---:|---:|---|
| Istio discovery at 1,000+ services | 6.0 | 8.7 | Generated dependency contracts, discovery selectors, xDS convergence evidence |
| Cilium / eBPF security | 5.0 | 9.2 | Separate network policy, flow observability, and runtime process enforcement |
| Multi-cloud routing, IAM, secrets | 6.5 | 8.3 | Regional autonomy, exact transit design, native workload identity |
| Intermittent systemd failures | 5.5 | 9.1 | Rate-limited node repair with fleet-level circuit breakers |
| Custom AMI qualification | 5.0 | 8.8 | Versioned certification suite, kernel and Kubernetes compatibility gates |
| Business-aware probes | 4.5 | 9.5 | Restart only when restart can help; synthetic monitoring for user journeys |
| DNS failure in a mesh | 5.0 | 8.8 | Classify DNS paths and preserve degraded service where possible |
| Terraform backend outage | 5.0 | 9.5 | One lineage, one backend, one writer |
| Envoy mTLS RCA | 7.0 | 9.1 | Inspect effective policy and exact failing hop |
| HPA not scaling | 7.5 | 9.5 | Separate HPA, scheduler, and node-autoscaler control loops |
| Cache sidecar P99 regression | 6.5 | 9.0 | Trace-based latency decomposition and falsifiable bypass experiment |
| Playback 504 | 4.5 | 9.5 | Identify the timeout emitter; health checks do not prove playback |
| NAT cost surge | 7.5 | 9.1 | Billing evidence first, then Flow Logs and workload attribution |
| SLO ownership | 7.0 | 9.1 | Joint product policy and graduated consequences |
| Multi-region, no DNS | 4.0 | 7.8 | Challenge impossible constraints; routing does not solve state |
| Chaos for a major release | 8.0 | 9.4 | Business invariants, cells, retry storms, and abort controls |
| Modernization ROI | 8.0 | 9.4 | Realized value, defensible causality, and no double counting |

---

# Round 1 - Systems at Scale, Kubernetes, Cloud, and Linux

## 1. Fine-grained service discovery across 1,000+ services

### Board ruling

The correct framing is a **configuration-distribution and dependency-boundary problem**, not merely DNS or service registry lookup.

At this scale, measure:

- clusters, listeners, routes, and endpoints per proxy;
- Envoy memory and startup time;
- xDS push and convergence time;
- ACK/NACK rates;
- stale proxy population;
- endpoint churn;
- control-plane CPU and queue depth.

Use multiple layers of scoping:

1. `discoverySelectors` to restrict which namespaces a control plane watches.
2. `exportTo` to limit resource visibility.
3. `Sidecar.egress.hosts` to constrain imported service configuration for sidecar-mode workloads.
4. Mesh or control-plane partitions when business domains or trust boundaries justify them.
5. East-west gateways for controlled cross-domain or cross-cluster communication.

Do not manually maintain 1,000 hand-written dependency lists. Use a service catalog or platform contract:

```text
service metadata / declared dependencies
        -> validation in CI
        -> generated mesh scoping configuration
        -> comparison against observed flows
        -> warn mode
        -> canary enforcement
        -> broad enforcement
```

`REGISTRY_ONLY` is traffic governance, not the primary security boundary. Network policy, authorization policy, identity, and egress controls remain necessary.

### Top-1% answer

> I first measure how much configuration each proxy receives and how quickly it converges. I then reduce fan-out using mesh-wide namespace scoping, resource visibility controls, and workload-specific dependency views. The platform generates those views from a service catalog, validates them against observed traffic, and canaries enforcement. Success is not that Istiod knows about 1,000 services; it is that each workload receives a small, correct, authorized view and continues serving from its last accepted configuration during control-plane disruption.

### Measurable acceptance examples

- Proxy P99 startup below the agreed deployment SLO.
- xDS convergence below the agreed threshold after a normal endpoint change.
- No unexplained NACKs.
- Material reduction in clusters/routes and proxy memory.
- No undeclared dependency outage during canary enforcement.

## 2. eBPF and Cilium

### Board ruling

Separate four capabilities:

- Cilium networking and service load balancing.
- Cilium L3/L4 and selective L7 network policy.
- Hubble flow observability.
- Tetragon or another runtime security layer for process, file, and syscall enforcement.

Do not present the decision as "Cilium versus Calico equals eBPF versus iptables." Calico also has an eBPF dataplane. Compare actual dataplane modes and operational requirements.

Do not claim all L7 policy executes purely inside eBPF. L7 enforcement may use Envoy. Do not promise universal performance improvement without measurement.

### Migration gates

- Kernel and eBPF feature compatibility.
- MTU and encapsulation behavior.
- kube-proxy replacement correctness.
- BPF map pressure and verifier failures.
- Encryption overhead.
- Service mesh interoperability.
- Policy equivalence.
- DNS latency, retransmissions, packet loss, and CPU per packet.

### Top-1% answer

> I choose Cilium for identity-aware dataplane control and high-fidelity flow evidence, not because eBPF is fashionable. I canary it in a dedicated node group, compare policy outcomes and network SLIs against the current dataplane, and keep runtime process enforcement as a separate Tetragon control. The migration has explicit rollback thresholds for packet loss, retransmissions, DNS latency, map pressure, and policy errors.

## 3. Cross-cloud routing, IAM, and secrets

### Board ruling

Start with invariants:

- regional service continues during loss of the inter-cloud link;
- no transitive route leak;
- no long-lived workload credentials;
- compromise in one cloud does not grant broad access in another;
- secret delivery is not a single globally synchronous dependency;
- remote calls have bounded retries, timeouts, and concurrency.

### Routing design questions

Specify:

- who owns the middle-mile network;
- where BGP terminates;
- which prefixes are advertised;
- route filtering and maximum-prefix controls;
- symmetric return paths;
- hub-and-spoke, cloud WAN, provider backbone, colo transit, or SD-WAN design;
- behavior during slow or partially degraded links;
- service-level gateways versus arbitrary pod-CIDR reachability.

Keep the normal request path local. Cross clouds through controlled gateways with circuit breakers and graceful degradation.

### Workload identity

Use native identity in each cloud:

- AWS: IRSA or EKS Pod Identity.
- Azure: Azure Workload Identity.
- GCP: Workload Identity Federation.

Then use narrowly scoped federation at the trust boundary. Restrict issuer, subject, audience, namespace, service account, session duration, and target role. Prevent fallback to broad node credentials.

### Secrets

Use a governance authority with a regional serving path:

- ESO when a Kubernetes Secret is acceptable.
- CSI or agent-mounted files when Kubernetes Secret persistence should be avoided.
- Native application retrieval for dynamic credentials and lease renewal.
- Regional Vault clusters or cloud secret stores with tested isolation and replication behavior.

### Top-1% answer

> My goal is common policy, not a single global dependency. Routing, identity, and secret delivery remain regionally autonomous. Cross-cloud access occurs through explicit gateways and narrowly scoped federation, and the application has a defined degraded mode when the inter-cloud path is slow or unavailable.

## 4. Intermittent systemd failures on EKS nodes

### Board ruling

The Descheduler is not a node-repair controller. Use node health monitoring plus a dedicated remediation mechanism, such as managed node repair or a carefully designed controller.

The response must distinguish:

- kubelet failure;
- container runtime failure;
- CNI failure;
- CSI/storage failure;
- filesystem/inode exhaustion;
- kernel OOM, panic, or driver failure;
- API, certificate, DNS, or IMDS connectivity.

### Repair state machine

1. Detect repeated or sustained degradation.
2. Correlate by AMI, kernel, instance family, Availability Zone, and rollout.
3. Mark degraded and cordon.
4. Confirm replacement capacity and disruption-budget feasibility.
5. Drain safely.
6. Reboot or replace according to failure class.
7. Preserve a sample node and host logs for RCA.
8. Rate-limit repair.
9. Stop automation if fleet health crosses a safety threshold.

### Top-1% answer

> An individual node is disposable; fleet capacity is not. Automated repair must have a circuit breaker so it cannot terminate nodes faster than capacity can be restored. I preserve enough evidence to distinguish a single bad node from a systemic AMI, kernel, CNI, or Availability Zone problem.

## 5. Custom AMI qualification

### Board ruling

A successful boot is not qualification. Promote only after a versioned certification suite.

### Qualification layers

**Supply chain**

- Pin and approve the parent image.
- Verify package sources and signatures.
- Generate an SBOM.
- Scan packages and filesystem content.
- Sign and attest the image.
- Restrict distribution permissions.

**Host and kernel**

- Cgroup mode.
- Seccomp.
- AppArmor or SELinux.
- Required eBPF features and BPF JIT policy.
- Conntrack and netfilter behavior.
- ENA/NVMe drivers.
- Filesystem, inode, and mount options.
- Time synchronization and entropy.
- FIPS requirements where applicable.

**Kubernetes integration**

- Node registration and readiness time.
- CNI and DNS.
- CSI mounts and EBS attachment.
- Image pulls.
- Kubelet and runtime restart.
- Drain, eviction, and graceful shutdown.
- Disk pressure and reboot recovery.
- Credential, logging, and monitoring agents.

**Promotion**

- Disposable cluster.
- Non-production canary node group.
- Small production node group.
- One zone at a time.
- Broad fleet after acceptance criteria pass.

### Top-1% answer

> I promote an AMI because it passes a reproducible certification suite and survives representative workloads and failure injection, not because it boots and joins the cluster.

## 6. Business-aware Kubernetes probes

### Board ruling

This is a frequent interview trap.

- Liveness: will restarting this container probably help?
- Readiness: should this instance receive new traffic now?
- Startup: has initialization completed?

A shared database, Kafka cluster, cache, or external API outage should not normally cause liveness failure. Restarting every pod can destroy caches, amplify connection storms, and remove degraded functionality that still helps customers.

Even readiness should represent **minimum serving capability**, not perfection across the entire dependency graph.

Use separate controls:

- kubelet probes for container and endpoint lifecycle;
- circuit breakers for dependency protection;
- synthetic transactions for business journeys;
- SLIs for customer outcomes;
- custom metrics for autoscaling;
- cached, bounded, and jittered dependency checks.

### Top-1% answer

> Kubelet probes are lifecycle controls, not end-to-end business monitoring. Liveness proves internal forward progress and fails only when restart can help. Readiness proves minimum serving capability. A real playback, checkout, or entitlement journey is validated by synthetic monitoring and customer-centered SLIs.

## 7. DNS outage inside a service mesh

### Board ruling

EDS does not make DNS irrelevant. Classify the path:

1. Application lookup before interception.
2. CoreDNS service and pod health.
3. Node-local DNS cache.
4. Upstream recursive resolver.
5. UDP fragmentation, TCP fallback, conntrack, or packet loss.
6. Kubernetes services, headless services, and external names.
7. ServiceEntry and mesh DNS capture behavior.
8. Negative caching and TTL behavior.

Existing Envoys may continue routing to known endpoints, but applications may still need DNS for new connections or external names.

### Mitigation without redeploying the application

- Restore or scale CoreDNS.
- Correct CoreDNS configuration or upstream resolvers.
- Repair kube-dns service routing.
- Restore NodeLocal DNSCache.
- Apply a controlled ServiceEntry or egress-gateway route where appropriate.
- Use existing mesh DNS capture if the architecture already supports it.
- Hardcoded addresses only as an owned, expiring incident bridge.

### Top-1% answer

> I first determine which names and workloads fail and whether established connections continue. Then I isolate application DNS, cluster DNS, upstream resolution, and mesh endpoint distribution. I restore the smallest failing layer and avoid converting a DNS incident into a stale-IP configuration problem.

## 8. Terraform backend timeout

### Board ruling

The invariant is:

> One state lineage, one authoritative backend, one writer.

### Recovery sequence

1. Freeze CI and local writers.
2. Determine whether an apply remains active.
3. Preserve logs, lock ID, run ID, plan, provider versions, and emergency local state.
4. Restore backend reachability, permissions, KMS, and throttling.
5. Pull and back up the authoritative state.
6. Inspect object versions, serial, and lineage.
7. Prove there is no active writer.
8. Remove a stale lock only when that proof exists.
9. Run a normal or refresh-only plan after backend recovery.
10. Resume one writer and observe.

Do not use `-lock=false` to move faster. Use `-lock-timeout` for normal contention. Backend migration must be explicit and controlled; a second bucket must not become an accidental split-brain authority.

### Top-1% answer

> State integrity is more important than deployment velocity. I recover the authoritative backend before making infrastructure changes unless a documented DR procedure explicitly transfers state authority. Force-unlock and state push are exceptional, reviewed actions with backups and lineage validation.

---

# Round 2 - RCA, Fire Drills, and Large-Scale Chaos

## 1. Envoy rollout broke mTLS

### Board ruling

Do not begin by disabling mTLS. Identify which hop changed its security expectation.

Inspect:

- rollout revision and affected scope;
- `istioctl proxy-status` and xDS ACK/NACK state;
- effective listeners, clusters, endpoints, and secrets;
- client-side `DestinationRule` and auto-mTLS behavior;
- server-side `PeerAuthentication`;
- gateway termination versus passthrough;
- SDS certificate presence and expiry;
- trust domain, SAN, SNI, and port protocol;
- sidecar revision skew and stale configuration;
- Envoy access-log response flags and TLS errors.

The trigger may be configuration mismatch, double TLS origination, missing secrets, expired certificates, trust-domain mismatch, or route/port mistakes.

### Top-1% answer

> I stop the rollout, compare working and failing revisions, and trace one connection from the client-side cluster to the server-side listener. I inspect the effective proxy configuration rather than the YAML I expect to be active. I roll back the smallest relevant change and validate both connectivity and the intended security policy.

## 2. HPA does not scale although Prometheus shows CPU above 80%

### Board ruling

Prometheus is a clue, not necessarily the metric source used by HPA.

Split the problem into three control loops:

1. Did HPA calculate a higher desired replica count?
2. Did the workload controller create those replicas?
3. Could the scheduler place them, and could the node autoscaler create capacity?

Inspect:

- HPA conditions: `AbleToScale`, `ScalingActive`, `ScalingLimited`.
- Metric source: resource, custom, external, or container resource.
- CPU requests on the targeted container.
- Metric freshness and selector correctness.
- Tolerance, stabilization windows, and scaling policies.
- `maxReplicas`.
- Pending pods, quotas, affinity, taints, topology constraints, storage, and IP exhaustion.
- Node autoscaler quotas and instance-type availability.

### Top-1% answer

> I do not diagnose this from a Prometheus graph alone. I identify the exact metric HPA consumed, how it normalized the value, the replica count it computed, and what happened to those replicas through the Deployment, scheduler, and capacity-control loops.

## 3. Cache sidecar creates P99 latency

### Board ruling

Tail latency is not automatically resource contention. Decompose one traced request:

```text
application queue
-> sidecar queue
-> cache lookup
-> serialization
-> cache miss
-> backend call
-> retry
-> response transfer
```

Compare hit and miss paths separately. Correlate by key, object size, node, zone, version, and request class.

Hypotheses include:

- CPU throttling or single-thread saturation.
- GC, page reclaim, or eviction storms.
- Hot-key locks.
- Synchronized TTL expiry and cache stampede.
- Large-value amplification.
- Connection-pool or file-descriptor exhaustion.
- Retry amplification.
- Backend slowdown exposed by miss bursts.

### Falsifiable experiment

Route a small percentage of equivalent traffic around the sidecar. If P99 improves and backend load remains within limits, the sidecar path is isolated rather than merely correlated.

### Mitigations

- Percentage bypass or rollback.
- TTL jitter.
- Request coalescing.
- Stale-while-revalidate or stale-on-error where safe.
- Negative caching.
- Object-size limits.
- Retry budgets.
- Correct CPU and memory reservations.

## 4. Playback returns 504 while health checks are green

### Board ruling

A 504 means a gateway or proxy exhausted a timeout while waiting on another hop. It does not prove that networking or the mesh is healthy.

First identify the emitter:

- CDN.
- Cloud load balancer.
- Ingress or edge gateway.
- Mesh sidecar.
- Application gateway.

Then correlate request/trace ID and inspect:

- Envoy response flags and response-code details.
- Route, per-try, and stream-idle timeout.
- Upstream and downstream duration.
- Whether headers or response bytes began flowing.
- Retry count and retry budget.
- Connection-pool pending requests and resets.
- Application threads/goroutines and dependency pools.
- Manifest, entitlement, DRM/license, metadata, cache, origin, and object-store latency.

### Top-1% answer

> A green health check proves the health-check contract. It does not prove the playback contract. I identify the exact timeout emitter, trace a real long-lived playback request, and determine whether the delay is queueing, application work, dependency saturation, retry amplification, packet loss, or a timeout that is incompatible with streaming semantics.

## 5. NAT cost surge without infrastructure changes

### Board ruling

Begin with billing and traffic evidence.

1. Confirm line item, region, NAT Gateway, and time window.
2. Compare NAT byte and connection metrics.
3. Attribute traffic through Flow Logs to source subnet, ENI, node, and workload.
4. Group bytes by destination and resolve the service or provider.
5. Correlate with workload deployment, retries, backup, logging, and DNS behavior.
6. Check route-table and VPC endpoint configuration.
7. Evaluate cross-AZ traffic.
8. Treat unexplained egress as a possible security event.

Common causes:

- crash-loop image pulls;
- public registry fallback;
- endpoint or route detachment;
- telemetry cardinality explosion;
- backup/replication job;
- public instead of private service endpoint;
- retry storm;
- large model or media downloads;
- cross-AZ NAT use;
- compromised workload or exfiltration.

Do not create interface endpoints blindly. Compare NAT, endpoint, AZ, and operational cost using actual volume.

---

# Round 3 - Leadership, Chaos Culture, and Engineering Influence

## 1. Building SLO ownership

### Board ruling

A universal automatic deployment freeze is too simplistic. Error-budget policy must be jointly agreed by product, engineering, and reliability stakeholders.

A customer-critical service needs:

- named product and engineering owners;
- customer-centered SLIs;
- an SLO and measurement window;
- error-budget dashboard;
- multi-window burn-rate alerts;
- a runbook and dependency map;
- capacity and graceful-degradation design;
- explicit consequences for budget consumption.

Use graduated consequences:

- normal burn: normal delivery;
- elevated burn: investigation and stricter canarying;
- fast burn: stop risky rollout;
- exhausted budget: prioritize reliability work;
- security and reliability fixes remain permitted;
- business exceptions require explicit, time-bounded risk acceptance.

Avoid hundreds of unused microservice SLOs. Start with user journeys: login, discovery, playback start, playback continuity, entitlement, and DRM.

### Top-1% answer

> SLO ownership exists when the SLO changes product decisions. Product and engineering jointly define the user journey, SLI, target, and error-budget policy. Reliability is then a visible business trade-off rather than an operations complaint.

## 2. Multi-region failover in three weeks with no DNS

### Board ruling

Challenge the premise before choosing a tool.

Ask:

- Does no DNS mean no DNS change or no DNS dependency?
- Must the existing public IP remain unchanged?
- Is there an existing global edge or anycast provider?
- Is the system single-cloud or multi-cloud?
- Is portable address space already owned?
- Can clients receive dynamic endpoint configuration?
- What are RTO, RPO, data consistency, and session requirements?
- Is the secondary region already deployed and sized?

If there is only one regional IP, no DNS change, no global edge, no portable prefix, and no client fallback, transparent Internet-wide failover may be impossible without changing the client-visible endpoint. A Principal engineer states this rather than inventing a magical BGP solution.

### Decision tree

**Existing global edge / CDN / anycast provider**

Add the second region as an origin, implement health-aware routing, and canary traffic.

**AWS-only**

AWS Global Accelerator may provide static anycast addresses and health-based routing to supported AWS endpoints. It still does not solve data, state, dependency, or capacity readiness.

**Multi-cloud**

Use an existing provider-independent edge, CDN, L4 proxy, or owned global network that can reach both clouds. Do not build a new Internet BGP platform in three weeks.

**Existing client fallback**

Use it as a secondary path. Do not assume a new client release reaches the full installed base in three weeks.

### Real work

- Data replication and lag.
- Write fencing or conflict semantics.
- Idempotency and duplicate messages.
- Session reconstruction.
- Regional IAM, secrets, and dependencies.
- Cache warming.
- Full-load capacity.
- Traffic dial and rollback.
- Failback.

### Top-1% answer

> Routing is the easiest part. I do not call the system multi-region until the surviving region can absorb production load and preserve the required data semantics. A second healthy load balancer is not a disaster-recovery system.

## 3. Chaos and graceful degradation for a major release

### Board ruling

Begin with business invariants:

- authentication succeeds;
- entitlement and DRM succeed;
- title discovery functions;
- playback starts;
- playback continues;
- nonessential personalization may degrade.

Test progressively:

1. Pod.
2. Node.
3. Dependency instance.
4. Cell or shard.
5. Availability Zone.
6. Regional dependency.
7. Inter-region path.
8. Full region evacuation.

Inject more than clean outages:

- latency and packet loss;
- resets and partial responses;
- DNS failure;
- CPU/memory/disk pressure;
- credential and certificate failure;
- queue backlog;
- cache loss;
- stale configuration;
- retry storm;
- database failover;
- surviving-region cold-cache and connection storms.

Every experiment needs:

- written hypothesis;
- steady-state metric;
- narrow initial blast radius;
- automated abort condition;
- incident commander;
- rollback or kill switch;
- capacity headroom;
- post-experiment actions.

Separate Tier-0 playback/authentication resources from lower-tier recommendations and enrichment using pools, bounded queues, retry budgets, circuit breakers, and load shedding.

### Top-1% answer

> The experiment passes only if a lower-tier dependency can fail without consuming Tier-0 capacity through retries, shared queues, worker pools, or connection limits.

## 4. Proving modernization ROI

### Board ruling

Do not sell newer tools. Sell measurable reduction in unit cost, delivery delay, or expected loss.

Measure:

**Velocity**

- lead time for change;
- deployment frequency;
- environment provisioning time;
- developer wait time;
- platform-ticket dependency;
- service onboarding time.

**Risk**

- incident frequency;
- customer-impact minutes;
- MTTR;
- change failure rate;
- recovery-test success;
- compliance findings;
- defensible expected annual loss.

**Margin**

- cost per request, stream, workload, or customer;
- idle-resource percentage;
- NAT and cross-zone spend;
- license retirement;
- toil and on-call hours;
- realized savings rather than theoretical recommendations.

Use conservative, expected, and upside cases. State assumptions, confidence, adoption dependency, and time to value.

```text
Net annual value =
    realized infrastructure savings
  + measurable delivery value
  + defensible avoided-loss estimate
  + retired license/support cost
  - ongoing platform operating cost
```

```text
Payback period =
  one-time implementation cost / monthly realized net benefit
```

Avoid false savings: recovered engineering time becomes financial value only if it avoids hiring, reduces external spend, accelerates measurable delivery, or is redirected to valued work. Avoid double-counting overlapping outage, revenue, retention, and support impacts.

### Top-1% answer

> I establish a baseline, run a pilot, measure adoption and realized outcomes, and stop or change direction when the assumptions are not validated. Infrastructure modernization is a business program with technical mechanisms, not an architecture fashion project.

---

# Cross-cutting Principal-level rules

## 1. Begin with invariants, not tools

Weak:

> I would deploy Cilium, Vault, and Global Accelerator.

Strong:

> The invariants are workload identity without static credentials, bounded failure domains, no transitive route exposure, continued regional operation during inter-cloud loss, and a tested rollback path.

## 2. State the failure model

For each design, name:

- complete, slow, inconsistent, and partial failure;
- blast radius;
- detection time;
- RTO and RPO;
- degraded mode;
- recovery ownership;
- failback behavior.

## 3. Separate control plane and data plane

Examples:

- Istiod may be unavailable while existing Envoys serve the last ACKed configuration.
- HPA may request replicas while the scheduler cannot place them.
- Terraform backend failure does not stop running infrastructure.
- CoreDNS failure does not necessarily terminate established connections.
- Load-balancer health checks can pass while the real customer path fails.

## 4. Treat partial failure as the normal hard case

Expect:

- 2% packet loss;
- one slow Availability Zone;
- 5% stale proxies;
- one inconsistent certificate chain;
- one overloaded shard;
- one client generation with different timeout behavior;
- retry traffic that appears healthy while amplifying overload.

## 5. Put limits on automation

Every remediation controller needs:

- rate limit;
- blast-radius limit;
- fleet-health threshold;
- audit trail;
- stop condition;
- approval boundary where appropriate;
- rollback or kill switch.

## 6. Quantify acceptance

Use system-specific targets. Examples:

- proxy P99 startup and xDS convergence targets;
- maximum node-repair percentage per interval;
- failover RTO and data RPO;
- zero state-lineage divergence;
- playback-start SLO maintained during recommendation failure;
- P99 improvement required from a cache sidecar;
- realized NAT-cost reduction verified after rollout.

---

# Approved interview frameworks

## REQUIRE - architecture and design

1. **R - Requirements and customer outcome.**
2. **E - Error budget, RTO, RPO, and consistency.**
3. **Q - Quantified scale and workload shape.**
4. **U - Untrusted assumptions and failure domains.**
5. **I - Implementation options and trade-offs.**
6. **R - Rollout, rollback, operability, and ownership.**
7. **E - Evidence and acceptance criteria.**

### Architecture opening

> Before selecting a product, I would clarify the customer outcome, scale, latency, consistency, RTO/RPO, trust boundaries, and what must continue during control-plane or regional failure.

## STABILIZE - incidents and RCA

1. **S - Stop the blast radius.**
2. **T - Trace a representative customer request or control-loop decision.**
3. **A - Ask what changed and preserve evidence.**
4. **B - Bound the failure by region, zone, node, version, tenant, or request class.**
5. **I - Inspect authoritative signals at each layer.**
6. **L - Limit harm with the smallest safe mitigation.**
7. **I - Identify trigger, root cause, and contributing conditions.**
8. **Z - Zero out recurrence with guardrails and tests.**
9. **E - Explain impact, verification, and measurable prevention.**

### Incident opening

> I would first contain customer impact and preserve evidence. Then I would follow one representative failing request or control-loop decision end to end, isolate the failing layer, apply the smallest reversible mitigation, and validate recovery against the customer SLI.

---

# Red-flag phrases to avoid

- "The network is healthy because the load balancer is green."
- "A 504 means the application is deadlocked."
- "CoreDNS can fail and Envoy will handle everything."
- "I would force-unlock Terraform and continue."
- "Liveness checks the database so Kubernetes restarts bad pods."
- "Cilium blocks anomalous syscalls."
- "The Descheduler replaces unhealthy nodes."
- "We can implement global BGP Anycast in three weeks."
- "This architecture guarantees an 80% reduction in risk."
- "Prometheus shows 80%, therefore HPA is broken."

# High-signal phrases

- "Which component is authoritative for this decision?"
- "What evidence distinguishes correlation from causation?"
- "I separate mitigation from root cause."
- "I need to know which hop emitted the timeout."
- "I change one layer at a time."
- "The automation has a fleet-level circuit breaker."
- "The green health endpoint proves only its own contract."
- "One state lineage, one authoritative backend, one writer."
- "The second region is not ready until state, capacity, and dependencies are proven."
- "Success criteria are defined before rollout."

---

# Recommended practice sequence

1. Deliver each scenario in 90 seconds using REQUIRE or STABILIZE.
2. Repeat it in five minutes with concrete commands, metrics, and rollback criteria.
3. Answer three adversarial follow-ups:
   - Why not restart it?
   - Why not increase the timeout?
   - Why not bypass the security or locking control temporarily?
4. State one measurable success criterion.
5. Close with the durable prevention mechanism and ownership model.

A strong answer is concise at the top and deep when challenged. Do not dump every tool immediately. Lead with the engineering decision, then support it with implementation evidence.
