# Principal Platform Engineering Interview Curriculum

## Chapter 1 — Fine-Grained Service Discovery Across 1,000+ Microservices with Envoy or Istio

> **Original scenario:** Design fine-grained service discovery for more than 1,000 microservices using Envoy or Istio. Explain how you prevent configuration explosion, control-plane saturation, broad service visibility, and mesh-wide failure propagation.

---

## 1. Why this question exists

This is not primarily a DNS question and it is not a test of whether you remember Istio resource names.

The interviewer is testing whether you understand:

- Kubernetes service discovery from first principles.
- The difference between a registry, a control plane, and a data plane.
- How Istiod transforms Kubernetes and Istio objects into xDS resources.
- Why configuration distribution becomes expensive at large scale.
- How service visibility, dependency scope, and organizational boundaries affect scalability.
- How already-programmed proxies behave when the control plane is unavailable.
- How to introduce a service mesh without creating one enormous failure domain.
- Whether you can discuss sidecar and Ambient architectures without treating either as universally superior.
- Whether you define measurable success criteria rather than simply saying “deploy Istio.”

A weak answer explains that Kubernetes has Services and CoreDNS.

A senior answer explains Envoy, Istiod, xDS, and `Sidecar`.

A Principal-level answer explains the configuration graph, what grows with scale, how the graph is bounded, how churn propagates, what survives control-plane failure, and how the operating model prevents a local change from becoming a mesh-wide event.

---

# 2. The interview answer first

## 2.1 Ninety-second answer

> I would treat this as a configuration-distribution and failure-domain problem, not merely a DNS problem. Kubernetes Services and EndpointSlices remain the source of endpoint truth. Istiod watches the selected Kubernetes and Istio resources, computes the configuration required by each proxy, and distributes listeners, routes, clusters, endpoints, and identity material over xDS.
>
> The central rule is that a workload should receive only the services and policies it is authorized and expected to use. I would scope the mesh at three levels: `discoverySelectors` for namespaces Istiod should ignore, `exportTo` for producer-controlled visibility, and `Sidecar.egress.hosts` for consumer dependency imports. Those declarations should be generated or validated against a service catalog rather than maintained as thousands of unrelated hand-written objects.
>
> I would partition operational failure domains by region, environment, trust boundary, and business domain; prefer local endpoints; and avoid one global endpoint graph unless the application requires it. Istiod would run with revisioned canaries, topology spreading, disruption protection, capacity headroom, and metrics for push latency, queueing, endpoint churn, config bytes per proxy, reconnects, ACKs, NACKs, and stale configuration.
>
> Envoy must continue serving its last accepted configuration during a temporary Istiod outage. I would evaluate Ambient where shared L4 mTLS and identity reduce sidecar cost, while using waypoints only where L7 policy is required. Ambient changes the dataplane shape but does not remove xDS or control-plane scaling.
>
> My success criterion is not that Istiod can see 1,000 services. It is that each workload receives a small, correct, rapidly converging dependency view, local endpoint churn does not trigger mesh-wide recomputation, and a discovery-plane outage does not interrupt already-established service.

## 2.2 Fifteen-second executive summary

> Scope discovery by dependency and fault domain, keep the registry authoritative, reduce both Istiod processing and proxy configuration, preserve last-known-good dataplane behavior, and prove convergence under endpoint churn.

---

# 3. Assumptions to state before designing

An architecture answer becomes credible when assumptions are explicit.

Use a statement such as:

> I will assume the platform contains more than 1,000 Services, tens of thousands of pods, multiple clusters and regions, and that an individual workload normally depends on only a small subset of the total service graph. The design needs workload identity, mTLS, L7 traffic policy for selected paths, regional survivability, and continued operation during a temporary discovery-control-plane outage.

Questions you may clarify:

- Are the services in one cluster or many?
- How many regions and availability zones?
- Is the design greenfield or a migration?
- Is every service enrolled in the mesh?
- Is L7 policy required everywhere or only for selected paths?
- Are VMs or external services included?
- What is the maximum acceptable endpoint-convergence time?
- Is cross-region service-to-service traffic normal or exceptional?
- What are the tenancy and regulatory boundaries?
- What are the RTO and RPO?
- Are dependencies known from a catalog, code ownership, or observed traffic?

Do not spend the interview interrogating the interviewer for ten minutes. State two or three reasonable assumptions and proceed.

---

# 4. Foundations from the ground up

## 4.1 What service discovery actually means

Service discovery answers two related but different questions:

1. **What logical service should the caller use?**
2. **Which concrete network endpoint should receive this request now?**

A logical service may be:

```text
ledger.payments.svc.cluster.local
```

Its current endpoints may be:

```text
10.42.4.17:8080
10.42.8.23:8080
10.42.9.11:8080
```

Those pod addresses are temporary. Pods restart, scale, move, and disappear. The logical service identity must remain stable while its endpoint set changes.

Service discovery therefore includes:

- Naming.
- Endpoint registration.
- Endpoint removal.
- Health and readiness.
- Locality.
- Load balancing.
- Identity.
- Policy visibility.
- Configuration propagation.
- Failure behavior.

DNS can provide a name-to-address mapping, but it does not by itself provide the complete identity, route, policy, certificate, locality, subset, retry, timeout, or outlier-detection model expected from a service mesh.

## 4.2 Kubernetes Service

A Kubernetes `Service` creates a stable logical destination for a changing group of pods.

Typical elements:

- Stable DNS name.
- Stable virtual IP for a `ClusterIP` Service.
- A selector that identifies backing pods.
- One or more named ports.

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ledger
  namespace: payments
spec:
  selector:
    app: ledger
  ports:
    - name: http
      port: 8080
      targetPort: 8080
```

The Service does not itself contain every endpoint. Endpoint information is represented separately.

## 4.3 EndpointSlice

EndpointSlices represent the concrete backends associated with a Service.

They provide:

- Endpoint addresses.
- Readiness and serving conditions.
- Port information.
- Zone and topology information.
- A scalable alternative to one enormous Endpoints object.

Conceptual example:

```text
Service: payments/ledger
EndpointSlice A:
  10.42.4.17, zone-a, ready
  10.42.8.23, zone-b, ready

EndpointSlice B:
  10.42.9.11, zone-c, ready
```

When pods scale or change readiness, EndpointSlices change. At hyperscale, those changes form a continuous event stream.

This is where service discovery becomes a control-plane workload: every relevant endpoint mutation may require recalculating and distributing configuration.

## 4.4 CoreDNS and native Kubernetes routing

Without a service mesh, a typical request path is:

```text
Application
    |
    | resolves ledger.payments.svc.cluster.local
    v
CoreDNS
    |
    | returns Service ClusterIP
    v
Kernel / kube-proxy / eBPF service datapath
    |
    | selects backend
    v
Ledger pod
```

Depending on the CNI and cluster design, service translation may use:

- iptables,
- IPVS,
- eBPF,
- another load-balancing implementation.

The application typically knows the Service address, while the node datapath selects a backend.

---

# 5. How Istio changes the model

## 5.1 High-level architecture

```text
                 CONTROL PLANE

Kubernetes API ---------------------------+
 Services                                  |
 EndpointSlices                            |
 Pods                                      v
 Namespaces                             Istiod
 Nodes                         registry + config processing
 Istio resources                         xDS server
                                            |
                      ADS / xDS             |
                                            v
                 DATA PLANE

Application -> Envoy sidecar -> destination Envoy -> Application
```

In Ambient mode:

```text
Application pod
      |
      v
node-level ztunnel  ---- optional waypoint for L7 ---- destination ztunnel
```

Istiod still supplies discovery and configuration through xDS.

## 5.2 Registry, control plane, and data plane

### Registry

The registry contains service and endpoint truth.

Examples:

- Kubernetes Services and EndpointSlices.
- ServiceEntries.
- WorkloadEntries.
- VM registration.
- External or cloud service registries.

### Control plane

Istiod:

- Watches Kubernetes and Istio objects.
- Builds an internal model.
- Determines which configuration applies to which proxy.
- Distributes configuration over xDS.
- Supplies or coordinates workload identity and certificates through Istio’s security model.

### Data plane

Envoy, ztunnel, and waypoint proxies:

- Receive configuration.
- Establish traffic paths.
- Perform load balancing.
- Enforce policy.
- Handle TLS/mTLS.
- Emit telemetry.
- Continue using accepted configuration when the control plane is temporarily unavailable.

This separation is critical:

> Istiod decides configuration; Envoy serves traffic.

A control-plane outage does not necessarily imply immediate traffic failure. Already-programmed proxies normally continue with their last accepted state. However, they cannot learn about new endpoints, routes, policies, or certificates indefinitely.

---

# 6. xDS from first principles

xDS is a family of APIs used to configure Envoy dynamically.

## 6.1 LDS — Listener Discovery Service

A listener defines where Envoy accepts traffic and which filter chain processes it.

Questions LDS helps answer:

- Which ports should Envoy listen on?
- Is the connection HTTP, TCP, TLS, or another protocol?
- Which network or HTTP filters should run?

## 6.2 RDS — Route Discovery Service

Routes determine where matching HTTP requests are sent.

A route may match:

- Host.
- Path.
- Header.
- Method.
- Weight.
- Version subset.

Example concept:

```text
Host: playback.internal
Path: /v2/*
90% -> playback-v2
10% -> playback-v3
```

## 6.3 CDS — Cluster Discovery Service

An Envoy cluster is a logical upstream destination.

A cluster defines:

- Discovery type.
- Connection pool.
- Circuit breakers.
- TLS behavior.
- Load-balancing policy.
- Outlier detection.
- Reference to endpoint discovery.

## 6.4 EDS — Endpoint Discovery Service

EDS supplies concrete endpoints for a cluster.

Example:

```text
Cluster:
  outbound|8080||ledger.payments.svc.cluster.local

Endpoints:
  10.42.4.17:8080
  10.42.8.23:8080
  10.42.9.11:8080
```

When a pod is added, removed, or becomes unready, EDS may change.

At large scale, EDS churn is often one of the most important operational dimensions.

## 6.5 SDS — Secret Discovery Service

SDS supplies dynamic secret material such as:

- Workload certificates.
- Private keys.
- Trust bundles.

It is part of the identity and mTLS path, not merely routing.

## 6.6 ADS — Aggregated Discovery Service

ADS allows multiple xDS resource types to be coordinated over a common stream.

The important operational concepts are:

- Envoy connects to the xDS server.
- Istiod sends resources.
- Envoy validates them.
- Envoy ACKs accepted resources.
- Envoy NACKs rejected resources.
- Istiod tracks sent and acknowledged versions.

A `NACK` is not a cosmetic warning. It can indicate that the proxy rejected a configuration update and may still be running an older accepted configuration.

---

# 7. End-to-end endpoint update flow

Assume a new `ledger` pod becomes ready.

```text
1. Pod starts.
2. Readiness becomes true.
3. Kubernetes updates an EndpointSlice.
4. Istiod receives the watch event.
5. Istiod updates its internal service model.
6. Istiod computes which connected proxies are affected.
7. Istiod generates an endpoint update.
8. Istiod sends EDS over xDS/ADS.
9. Envoy validates the update.
10. Envoy ACKs or NACKs it.
11. The new endpoint becomes eligible for new requests.
```

Questions a Principal Engineer asks at every step:

- What is authoritative?
- What is cached?
- What is eventually consistent?
- What is the propagation delay?
- How many consumers receive the change?
- What if a consumer is disconnected?
- What if the update is invalid?
- What if thousands of endpoints change simultaneously?
- What if the same node repeatedly transitions between ready and unready?

---

# 8. Why scaling breaks

## 8.1 The naïve global graph

Let:

```text
S = number of services
E = number of endpoints
P = number of proxies
D = average dependencies per workload
```

If every proxy receives information about every service and endpoint, the distribution problem trends toward:

```text
P × S
```

or, for endpoint-heavy resources:

```text
P × E
```

The real implementation is more nuanced because configuration is shared, generated, cached, and incrementally delivered. The formula is still useful as a mental model: broad visibility makes every proxy participate in changes that may be irrelevant to it.

If each workload only requires `D` dependencies, the desired configuration relationship is closer to:

```text
P × D
```

where `D` is dramatically smaller than `S`.

The goal is not merely smaller YAML. The goal is to reduce:

- Istiod computation.
- Memory used by control-plane indexes and generated configuration.
- xDS bytes.
- Envoy memory.
- Proxy startup time.
- Reconnect cost.
- Churn fan-out.
- Organizational blast radius.

## 8.2 Endpoint churn

Common sources:

- Deployments.
- Autoscaling.
- Spot interruption.
- Node replacement.
- Readiness flapping.
- CNI failure.
- Availability-zone impairment.
- Batch workloads.
- Large cluster upgrades.

A single endpoint change is small. Thousands of correlated changes are not.

## 8.3 Node-flap amplification

```text
Node impairment
    |
    v
many pods become unready or disappear
    |
    v
many EndpointSlices change
    |
    v
Istiod receives burst of events
    |
    v
internal graph updates + affected-proxy calculation
    |
    v
xDS queueing and push delay
    |
    +--> proxy reconnect backlog
    +--> stale endpoint views
    +--> CPU saturation
```

The systemic danger is not only the failed node. It is that a local fault creates global control-plane work.

## 8.4 Proxy reconnect storms

After:

- Istiod restart.
- Network partition.
- Certificate problem.
- Mass pod restart.
- Sidecar upgrade.
- Cluster recovery.

Thousands of proxies may reconnect and request configuration at roughly the same time.

This may cause:

- CPU spikes.
- Memory pressure.
- Connection queueing.
- Large initial snapshots.
- Slow workload readiness.
- Inconsistent convergence times.

Capacity planning must test reconnect behavior, not only steady-state xDS pushes.

---

# 9. Configuration-scoping strategy

Current Istio documentation describes three complementary mechanisms:

1. `Sidecar` import.
2. `exportTo`.
3. `discoverySelectors`.

They operate at different ownership and processing layers.

## 9.1 `discoverySelectors`

`discoverySelectors` provide mesh-wide control over the namespaces Istiod considers.

Conceptual configuration:

```yaml
meshConfig:
  discoverySelectors:
    - matchLabels:
        istio-discovery: enabled
```

Selected namespace:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: payments
  labels:
    istio-discovery: enabled
```

### Purpose

- Exclude namespaces unrelated to the mesh.
- Prevent non-mesh workloads and configuration from adding unnecessary processing.
- Establish explicit enrollment.

### What it solves

- Control-plane processing of irrelevant namespaces.
- Accidental inclusion of build, monitoring, test, or system namespaces.
- Part of cluster-wide configuration-cost reduction.

### What it does not solve

- It does not define fine-grained dependencies within selected namespaces.
- It does not replace authorization.
- It does not make all selected configuration cheap.
- A namespace-labeling mistake can hide required services from Istio.

### Interview signal

> I use discovery selectors as the earliest mesh-wide boundary: do not ask Istiod to process namespaces that are not part of the mesh.

## 9.2 `exportTo`

`exportTo` allows the producer or resource owner to control visibility.

For a Kubernetes Service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ledger
  namespace: payments
  annotations:
    networking.istio.io/exportTo: ".,checkout"
spec:
  selector:
    app: ledger
  ports:
    - name: http
      port: 8080
```

Meaning:

- `"."` — own namespace.
- `"checkout"` — checkout namespace.
- Other supported visibility forms depend on the resource and current Istio API.

Istio resources such as `VirtualService`, `DestinationRule`, and `ServiceEntry` use `spec.exportTo`.

### Purpose

- Service-owner-controlled visibility.
- Reduce accidental global export.
- Express domain boundaries.
- Reduce configuration relevance.

### Important precision

Do not claim that `exportTo` alone means Istiod never watches the underlying Kubernetes object. Istiod’s watch behavior and internal filtering are separate from the resource’s visibility to workloads.

### Security warning

Visibility is not a complete authorization boundary.

Use:

- `AuthorizationPolicy`.
- Workload identity.
- Network policy.
- Egress enforcement.
- Gateway policy.

`exportTo` helps scope configuration and reduce accidental reachability, but it should not be the only control protecting a sensitive service.

## 9.3 `Sidecar.egress.hosts`

`Sidecar` allows workload owners to control imported configuration.

Namespace-wide example:

```yaml
apiVersion: networking.istio.io/v1
kind: Sidecar
metadata:
  name: default
  namespace: payments
spec:
  egress:
    - hosts:
        - "./*"
        - "identity/*"
        - "ledger/*"
        - "istio-system/*"
```

### Purpose

- Consumer-side dependency declaration.
- Reduce clusters, routes, and endpoints delivered to affected sidecars.
- Make workload dependencies explicit.

### Risks

- Rare or administrative traffic may be omitted.
- Manual declarations drift.
- Dynamic dependencies can be difficult to model.
- One wrong namespace-wide object can break many services.
- Teams may treat it as security while missing authorization controls.

### Operating model

At 1,000 services, do not expect teams to maintain dependency scope entirely by hand.

Use:

- Service catalog ownership.
- Dependency declarations in application metadata.
- Observed Hubble, mesh, or trace data.
- CI validation.
- Policy-as-code.
- Canary enforcement.
- Break-glass exceptions with expiration.
- Drift reporting.

A generated dependency contract is stronger than an undocumented manual list, but observed traffic must not automatically become permanent permission. Otherwise every accidental call expands the graph.

---

# 10. Sidecar and Ambient architectures

## 10.1 Sidecar mode

```text
Pod
+-----------------------+
| Application           |
| Envoy sidecar         |
+-----------------------+
```

Advantages:

- Mature per-workload L7 traffic management.
- Strong workload-level isolation.
- Fine-grained telemetry.
- Broad compatibility with existing Istio behavior.

Costs:

- CPU and memory per pod.
- Proxy startup and readiness.
- Large total proxy population.
- More xDS connections.
- Configuration duplication.
- Upgrade coordination.
- Application-proxy lifecycle coupling.

## 10.2 Ambient mode

```text
Pod traffic
    |
    v
ztunnel on node
    |
    +---- direct L4 path ----+
    |
    +---- waypoint for L7 policy/routing
```

### ztunnel

ztunnel provides shared node-level functions such as:

- L4 transport.
- Workload identity.
- mTLS.
- L4 policy and telemetry.

### waypoint

A waypoint proxy provides optional L7 functionality for selected traffic, such as:

- HTTP routing.
- L7 authorization.
- Retries and timeouts.
- Rich L7 telemetry.
- Circuit breaking.

### Benefits

- No sidecar resource tax in every pod.
- Application lifecycle less coupled to proxy injection.
- L4 security for workloads without requiring a per-pod Envoy.
- L7 proxies deployed only where needed.

### Risks and questions

- ztunnel remains a node-level failure domain.
- Waypoint capacity must be designed and measured.
- Policy attachment semantics differ from sidecar assumptions.
- Not every sidecar-era feature maps identically.
- Traffic capture, observability, upgrade, and rollback must be tested.
- Istiod and xDS still exist.
- Ambient does not eliminate control-plane processing.

### Principal conclusion

> Ambient is a dataplane architecture choice, not a universal scalability shortcut. I choose it when shared L4 identity and transport provide a better operational model and introduce waypoints only where L7 value justifies the additional hop and capacity domain.

---

# 11. Failure modes

## 11.1 Istiod unavailable

Expected:

- Existing proxies retain accepted configuration.
- Established traffic can continue.
- No new configuration can be distributed.
- New proxies may fail to become fully ready.
- New endpoints may not be learned.
- Removed endpoints may remain visible until other health mechanisms react.
- Certificate rotation eventually becomes a concern.

Interview statement:

> The data plane is designed to serve with last-known-good state, but that state has a freshness limit. I explicitly define how long endpoint, route, and certificate staleness is tolerable.

## 11.2 Invalid configuration and NACKs

Failure path:

```text
Bad route or cluster configuration
    |
    v
Istiod sends update
    |
    v
Envoy rejects resource
    |
    v
NACK / stale or inconsistent configuration
```

Investigate:

- Which resource type was rejected?
- Which configuration object generated it?
- Is the issue limited to one revision or namespace?
- Did Envoy preserve its previous accepted config?
- Are different proxies running different effective states?

## 11.3 EndpointSlice storm

Possible triggers:

- Autoscaler surge.
- Node group replacement.
- Readiness probe anti-pattern.
- Zone failure.
- Controller bug.

Controls:

- Stable readiness behavior.
- Hysteresis and debouncing where appropriate.
- Scoped configuration.
- Capacity headroom.
- Partitioned control planes or clusters where justified.
- Progressive node upgrades.
- Endpoint-convergence SLO.
- Fleet-level alerting on churn rate.

## 11.4 One globally exported service

A single high-churn Service exported mesh-wide can cause disproportionate configuration fan-out.

Controls:

- Default visibility policy.
- Admission checks.
- Ownership metadata.
- Config-cost analysis.
- Review for globally visible objects.
- Alerts on unexpected proxy config growth.

## 11.5 One giant mesh

Advantages:

- Unified identity.
- Shared policy model.
- Easier naming.
- Fewer gateways.

Risks:

- Large configuration graph.
- Organization-wide blast radius.
- Control-plane upgrades affect more teams.
- Policy mistakes propagate broadly.
- Troubleshooting and ownership become ambiguous.

## 11.6 Too many meshes

Advantages:

- Smaller failure domains.
- Independent upgrades.
- Clear ownership boundaries.

Risks:

- Cross-mesh gateways.
- Trust federation.
- Certificate and identity complexity.
- More operational control planes.
- Difficult cross-domain debugging.
- Duplicated policy and observability.

Principal rule:

> Partition where failure, trust, lifecycle, or ownership must be independent—not simply because the service count passed an arbitrary number.

---

# 12. Locality and endpoint policy

Prefer local traffic when it improves latency and reduces correlated network dependency, but do not blindly force all traffic to the same zone.

Questions:

- Is same-zone capacity sufficient after one-zone failure?
- Does strict locality strand healthy capacity elsewhere?
- What happens during a partial network partition?
- Are cross-zone charges material?
- Does the data dependency already require cross-zone access?
- Does outlier detection eject too many endpoints during correlated slowness?

Example `DestinationRule`:

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: ledger
  namespace: payments
spec:
  host: ledger.ledger.svc.cluster.local
  trafficPolicy:
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 5s
      baseEjectionTime: 30s
      maxEjectionPercent: 20
```

Do not present these values as universal. They must be derived from:

- Error distribution.
- Traffic volume.
- Endpoint count.
- Retry policy.
- Recovery time.
- Dependency saturation behavior.

Outlier detection can improve availability, but correlated errors can eject too much capacity. `maxEjectionPercent` and minimum healthy capacity matter.

---

# 13. Control-plane deployment design

## 13.1 High availability

Use:

- Multiple replicas.
- Topology spread.
- Pod anti-affinity where appropriate.
- Resource requests and limits.
- Priority class.
- Pod disruption controls.
- Capacity headroom.
- Load testing.
- Independent telemetry path.

## 13.2 Revisioned upgrades

Run a new control-plane revision and enroll canary workloads or namespaces.

Validate:

- Proxy connection.
- Config acceptance.
- Traffic SLI.
- Certificate issuance.
- Webhook behavior.
- Policy evaluation.
- Resource usage.
- Rollback.

Do not define success as “the Helm upgrade completed.”

Define success as:

- No regression in customer SLI.
- xDS convergence within target.
- No abnormal NACK or reconnect increase.
- Proxy memory within budget.
- Rollback proven.

## 13.3 Sharding and partitioning

Potential strategies:

- Separate clusters per region.
- Separate meshes by trust or business domain.
- Multiple Istiod deployments or revisions.
- External control plane where justified.
- Gateway-mediated cross-domain traffic.

Do not propose control-plane sharding casually. Explain:

- What is the shard key?
- How are proxies assigned?
- How is service visibility handled across shards?
- How is trust distributed?
- What happens if one shard is overloaded?
- How is operational ownership divided?

---

# 14. Observability

## 14.1 Control-plane signals

Monitor:

- Istiod CPU and memory.
- xDS push duration.
- Push queue depth or pending work.
- Total pushes.
- Incremental versus full updates.
- Connected proxies.
- Disconnect and reconnect rates.
- ACK and NACK counts.
- Config-generation errors.
- Kubernetes watch errors.
- Endpoint event rate.
- Certificate issuance failures.
- Admission webhook latency and errors.

## 14.2 Data-plane signals

Monitor:

- Envoy memory and CPU.
- Number of clusters, listeners, routes, and endpoints.
- Proxy startup time.
- xDS connection status.
- Stale resources.
- Upstream connection failures.
- No-healthy-upstream responses.
- Retry volume.
- Endpoint ejections.
- TLS errors.
- Request latency and saturation.

## 14.3 Product signals

Infrastructure telemetry is not enough.

Monitor:

- Successful user journeys.
- Request-class availability.
- Regional error rate.
- Playback/startup or checkout latency, depending on product.
- Dependency-specific SLOs.
- Degraded-mode usage.

A green Istiod dashboard does not prove that a customer request can complete.

---

# 15. Debugging path

## 15.1 Is the proxy connected and synchronized?

```bash
istioctl proxy-status
```

Interpretation:

- `SYNCED`: Envoy acknowledged the latest resource sent for that type.
- `STALE`: Istiod sent an update but has not observed acknowledgement.
- `NOT SENT`: Istiod did not send that resource type.
- Missing proxy: it is not currently connected to the queried Istiod view.

What it proves:

- xDS connection and acknowledgement state.

What it does not prove:

- Application correctness.
- End-to-end request success.
- Correct business routing.
- Healthy downstream service.

## 15.2 What clusters did Envoy actually receive?

```bash
istioctl proxy-config clusters <pod> -n <namespace>
```

Use filters where possible:

```bash
istioctl proxy-config clusters <pod> -n <namespace> \
  --fqdn ledger.payments.svc.cluster.local
```

Questions answered:

- Does the proxy know the upstream cluster?
- Which port and subset exist?
- What discovery type is used?
- Is TLS policy attached?

## 15.3 What endpoints does Envoy believe are available?

```bash
istioctl proxy-config endpoints <pod> -n <namespace>
```

Compare with Kubernetes truth:

```bash
kubectl get endpointslice -n payments \
  -l kubernetes.io/service-name=ledger -o yaml
```

If Kubernetes and Envoy differ, investigate:

- Propagation delay.
- xDS disconnect.
- Scoping.
- NACK.
- Incorrect Service or port naming.
- Readiness state.
- Locality filtering.
- Revision mismatch.

## 15.4 What routes did Envoy receive?

```bash
istioctl proxy-config routes <pod> -n <namespace>
```

Questions:

- Does the virtual host exist?
- Which route matched?
- Is traffic weighted?
- Is a timeout present?
- Is the expected cluster referenced?

## 15.5 What listeners exist?

```bash
istioctl proxy-config listeners <pod> -n <namespace>
```

Use when:

- The expected port is not captured.
- Protocol detection is wrong.
- Inbound and outbound behavior differ.
- Listener conflicts exist.

## 15.6 Compare Envoy and Istiod views

```bash
istioctl proxy-status <pod>.<namespace>
```

This can expose differences between what Istiod would generate and what Envoy has loaded.

## 15.7 Analyze configuration

```bash
istioctl analyze -A
```

Use before and during rollout, but remember:

- Static analysis catches known structural issues.
- It does not prove runtime dependency health.
- It does not replace canary traffic.

---

# 16. Incident response example

## Scenario

During a node-group update, playback errors increase. Istiod CPU reaches 100%, xDS pushes slow down, and some Envoys show stale EDS.

## STABILIZE response

### Stop the blast radius

- Pause node-group rollout.
- Freeze unrelated mesh changes.
- Stop aggressive autoscaling changes if they are amplifying churn.
- Assign incident ownership.

### Trace the user path

Follow one failed request:

```text
client
 -> regional edge
 -> ingress gateway
 -> playback API Envoy
 -> playback application
 -> metadata service
```

Identify:

- Which hop failed?
- Which component emitted the timeout or reset?
- Did the source Envoy have current endpoints?
- Did the destination exist and remain ready?

### Ask what changed

- Node-group rollout.
- EndpointSlice event rate.
- Istiod revision.
- New Sidecar or export policy.
- Readiness behavior.
- Traffic volume.

### Bound the failure

- Region.
- Cluster.
- Node group.
- Istiod revision.
- Namespace.
- Proxy generation.
- Endpoint type.

### Inspect authoritative signals

```bash
istioctl proxy-status
istioctl proxy-config endpoints <affected-pod> -n playback
kubectl get endpointslice -n playback
kubectl get events -A --sort-by=.lastTimestamp
kubectl top pods -n istio-system
```

Inspect control-plane metrics and logs for:

- push latency,
- queueing,
- reconnect rate,
- Kubernetes watch delay,
- NACKs,
- config generation errors.

### Limit harm

Potential mitigations:

- Pause the rollout generating churn.
- Scale Istiod if CPU is a proven bottleneck.
- Roll back a bad scoping/config revision.
- Shift traffic to an unaffected region or cluster.
- Reduce noncritical load.
- Disable a noncritical route or feature.
- Restore a known-good node or mesh revision.

Do not:

- Restart every sidecar.
- Increase every timeout.
- Disable mTLS mesh-wide.
- Create more endpoint churn without understanding the cause.

### Root and contributing causes

Example:

- Trigger: node-group replacement.
- Root systemic weakness: every proxy imported a globally visible high-churn service graph.
- Contributing factor: insufficient Istiod headroom and reconnect testing.
- Customer symptom: requests used stale endpoints and timed out.
- Repair risk: restarting all pods would create a larger reconnect storm.

### Permanent correction

- Reduce service visibility.
- Add discovery selectors.
- Generate consumer dependency scope.
- Test control-plane behavior during endpoint storms.
- Add push-latency and stale-proxy SLOs.
- Canary node and mesh changes independently.
- Establish fleet-level stop conditions.

---

# 17. Answers that sound correct but fail at scale

## “Kubernetes DNS solves service discovery.”

Incomplete. It solves name resolution but not the entire service mesh configuration, identity, routing, policy, and endpoint-distribution problem.

## “Set `REGISTRY_ONLY`.”

`REGISTRY_ONLY` controls behavior for unknown outbound destinations; it is not a complete security boundary and does not alone solve Istiod processing or configuration explosion.

## “Use `Sidecar` and the problem is solved.”

Sidecar import scope helps proxy configuration. It must be combined with producer visibility, mesh discovery scope, operational generation, validation, and control-plane capacity.

## “Use `exportTo`, so Istiod never sees the Service.”

Too absolute. Visibility and Kubernetes watch/processing behavior are related but not identical. `discoverySelectors` are the mesh-wide mechanism for ignoring excluded configuration early.

## “Ambient eliminates Envoy and xDS.”

Incorrect. Ambient uses ztunnel and optional waypoint proxies, and these still receive configuration from Istiod through xDS.

## “One mesh gives the strongest consistency.”

It may simplify some identity and policy concerns, but it increases shared blast radius and organizational coupling.

## “Use many meshes to solve scaling.”

It trades one scale problem for federation, gateway, trust, observability, and ownership complexity.

## “Retries protect users during endpoint churn.”

Retries may improve success for isolated failures, but they can amplify overload and consume concurrency. They require retry budgets and coherent timeout hierarchies.

## “If `proxy-status` is synced, traffic is healthy.”

It proves acknowledgement state, not end-to-end correctness.

---

# 18. Trade-off table

| Decision | Benefit | Cost or risk |
|---|---|---|
| Broad global visibility | Simple initial configuration | Large config, proxy memory, churn fan-out |
| `discoverySelectors` | Excludes irrelevant namespaces early | Enrollment mistakes can hide required objects |
| `exportTo` | Producer-controlled visibility | Not complete authorization; ownership discipline required |
| `Sidecar.egress.hosts` | Fine consumer dependency scope | Drift and rare-path breakage |
| Generated dependency contracts | Scalable governance | Catalog accuracy and platform ownership |
| One mesh | Unified identity and policy | Shared operational blast radius |
| Multiple meshes | Independent fault domains | Federation and gateway complexity |
| Sidecars | Mature workload-level L7 | Per-pod resource and lifecycle cost |
| Ambient | Shared L4 security and reduced sidecars | Different policy model and waypoint capacity |
| Locality preference | Lower latency and network cost | Can strand capacity or magnify zone imbalance |
| Outlier detection | Ejects unhealthy endpoints | Correlated failures can remove too much capacity |
| Delta xDS | Can reduce unnecessary update volume | Does not eliminate all recomputation or unchanged delivery |
| Control-plane sharding | Smaller processing domains | Assignment, trust, and cross-shard complexity |

---

# 19. Security analysis

Separate these concepts:

## Discovery visibility

Which services and configuration a proxy receives.

Controls:

- `discoverySelectors`.
- `exportTo`.
- `Sidecar.egress.hosts`.

## Authentication

Who the workload is.

Controls:

- Workload identity.
- Certificates.
- SPIFFE-like identities.
- mTLS.

## Authorization

Whether the caller may perform the requested operation.

Controls:

- Istio `AuthorizationPolicy`.
- Application authorization.
- API gateway policy.

## Network enforcement

Whether a network path is permitted.

Controls:

- Kubernetes NetworkPolicy.
- Cilium policy.
- Firewalls and security groups.
- Egress gateway controls.

## Routing behavior

Where allowed traffic is sent.

Controls:

- VirtualService.
- DestinationRule.
- Gateway routing.
- Locality and load balancing.

A Principal answer does not use service visibility as a substitute for authorization.

---

# 20. Rollout strategy

## Phase 1 — Measure

Collect:

- Clusters per proxy.
- Endpoints per proxy.
- Proxy memory.
- xDS payload and push timing.
- Endpoint churn.
- Current dependency graph.
- Rare and scheduled traffic.

## Phase 2 — Establish ownership

Every service receives:

- Owner.
- Business domain.
- Data classification.
- Allowed consumers.
- Environment.
- Criticality.
- Region.
- SLO.

## Phase 3 — Exclude irrelevant namespaces

Introduce discovery selectors with canary validation.

## Phase 4 — Producer visibility

Set default and explicit `exportTo` policies.

## Phase 5 — Consumer imports

Introduce generated or validated Sidecar dependency scope.

## Phase 6 — Progressive enforcement

Canary by:

- Workload.
- Namespace.
- Cluster.
- Region.

## Phase 7 — Failure testing

Test:

- Istiod outage.
- Proxy reconnect storm.
- EndpointSlice burst.
- Invalid config.
- Zone loss.
- Certificate rotation.
- Rollback.

## Rollback criteria

Roll back when:

- User SLI degrades.
- Proxy NACKs exceed threshold.
- Config convergence exceeds target.
- Unexpected denied or unknown dependencies appear.
- Proxy memory or CPU exceeds budget.
- New revision cannot sustain reconnect test.

---

# 21. Measurable acceptance criteria

Example targets must be derived from the environment, but the categories should be explicit.

- P99 endpoint convergence after a normal pod change.
- P99 convergence during a node-loss burst.
- Maximum xDS push duration.
- Maximum proxy initial sync time.
- Maximum number of clusters/endpoints per workload class.
- Envoy or ztunnel memory budget.
- Control-plane CPU headroom during steady state.
- Control-plane CPU headroom during reconnect storm.
- NACK rate.
- Stale proxy duration.
- Percentage of globally exported services.
- User-path availability during one Istiod replica loss.
- Already-established traffic behavior during complete temporary Istiod outage.
- Rollback completion time.
- Maximum allowed retry amplification.

Strong close:

> My rollout is complete only when the scoped design remains correct during rare traffic, converges within SLO during endpoint churn, survives control-plane loss with last-known-good behavior, and has a proven rollback path.

---

# 22. Whiteboard exercise

Draw this from memory:

```text
                           +------------------+
                           | Kubernetes API   |
                           | Services         |
                           | EndpointSlices   |
                           | Istio resources  |
                           +--------+---------+
                                    |
                                    | watches selected namespaces
                                    v
                          +--------------------+
                          | Istiod             |
                          | registry model     |
                          | config generation  |
                          | xDS / cert control |
                          +-----+---------+----+
                                |         |
                         xDS    |         | xDS
                                v         v
                       +-----------+   +-----------+
                       | Proxy A   |   | Proxy B   |
                       | scoped    |   | scoped    |
                       | config    |   | config    |
                       +-----+-----+   +-----+-----+
                             |               |
                             +------TLS------+
```

Add three scoping labels:

```text
discoverySelectors -> what Istiod considers
exportTo           -> what producers expose
Sidecar import     -> what consumers import
```

Then explain:

- What happens when the Kubernetes API is delayed?
- What happens when Istiod is down?
- What happens when Envoy NACKs?
- What happens when a node flaps?
- What changes in Ambient mode?

---

# 23. Adversarial interviewer follow-ups

## How does `exportTo` differ from `Sidecar.egress.hosts`?

`exportTo` is producer-controlled visibility. `Sidecar.egress.hosts` is consumer-controlled import scope. They can be used together.

## Why also use `discoverySelectors`?

Because it is the mesh-wide mechanism for excluding namespaces and configuration from Istio’s discovery processing. It reduces irrelevant control-plane work earlier than per-proxy import decisions.

## What happens if a client calls a service outside its imported scope?

Behavior depends on mesh configuration, traffic capture, service resolution, and outbound policy. I would not depend on accidental fall-through. Dependency scope must be tested, and security must be enforced separately through authorization and network policy.

## Does Envoy stop serving when Istiod dies?

Normally, already-configured Envoys continue using accepted configuration. They cannot receive new endpoint, route, policy, or certificate updates while disconnected.

## How long can the mesh operate without Istiod?

There is no universal duration. It depends on:

- Endpoint churn.
- Route changes.
- New workload starts.
- Certificate lifetime and rotation.
- Dependency health.
- Whether removed endpoints remain reachable.

The platform must define a stale-config tolerance.

## Why not deploy one Istiod per namespace?

It would create severe operational, trust, gateway, upgrade, and ownership complexity. Partition by real failure and trust boundaries, not by arbitrary namespace count.

## How do you maintain thousands of Sidecar objects?

Generate or validate them from service ownership and dependency metadata, augment with observed flows, canary them, and require CI declaration for new dependencies.

## Why not learn dependencies entirely from observed traffic?

Observed traffic is incomplete and can include accidental or malicious paths. It is evidence, not automatic authorization. Rare paths, disaster recovery, cron jobs, and maintenance traffic must be included deliberately.

## Does Ambient solve control-plane saturation?

It can reduce per-pod sidecar population and some configuration duplication, but ztunnels and waypoints still receive xDS state. Istiod processing and configuration scope remain relevant.

## How do you prevent retries from amplifying an outage?

Use:

- Retry budgets.
- Small bounded retry counts.
- Per-try timeouts.
- Overall request deadlines.
- Jitter.
- Circuit breakers.
- Load shedding.
- Idempotency.
- One clear retry layer where possible.

---

# 24. Hands-on lab

## Objective

Measure how configuration scope changes what an Envoy proxy receives.

## Prerequisites

- Kubernetes cluster.
- Current supported Istio release.
- `kubectl`.
- Matching `istioctl`.
- Three namespaces: `payments`, `identity`, `unrelated`.

## Lab stages

### Stage 1 — Baseline

Deploy two services in each namespace.

Inspect a payments proxy:

```bash
istioctl proxy-config clusters <payments-pod> -n payments
istioctl proxy-config endpoints <payments-pod> -n payments
```

Record:

- Number of clusters.
- Number of endpoints.
- Proxy memory.
- Initial sync time.

### Stage 2 — Add producer visibility

Annotate the ledger Service:

```yaml
metadata:
  annotations:
    networking.istio.io/exportTo: ".,identity"
```

Reinspect proxy configuration.

### Stage 3 — Add Sidecar scope

Create a namespace-wide `Sidecar` in payments.

Verify:

- Required dependencies remain.
- Unrelated clusters disappear.
- Rare paths still function.

### Stage 4 — Add discovery selectors

Label only mesh namespaces and configure selectors in a test control-plane revision.

Validate:

- Istiod ignores excluded namespaces.
- Required mesh resources remain visible.
- No unexpected errors appear.

### Stage 5 — Endpoint churn

Scale a deployment repeatedly:

```bash
kubectl scale deployment ledger -n payments --replicas=20
kubectl scale deployment ledger -n payments --replicas=3
```

Observe:

- EndpointSlice changes.
- xDS push duration.
- Proxy convergence.
- User request success.

### Stage 6 — Control-plane outage

In a nonproduction lab, temporarily stop Istiod.

Test:

- Existing requests.
- New connections to known endpoints.
- New pod startup.
- Endpoint changes.
- Recovery and reconnect behavior.

### Stage 7 — Invalid configuration

Apply a deliberately invalid or conflicting routing resource in a disposable namespace.

Observe:

- `istioctl analyze`.
- Istiod errors.
- Proxy ACK/NACK state.
- Last-known-good behavior.

## Lab report questions

1. How many clusters did the proxy have before and after scoping?
2. Did Envoy memory change?
3. Which change reduced control-plane processing versus proxy import?
4. What continued working during Istiod outage?
5. What did not?
6. How long did reconvergence take?
7. What rare traffic broke?
8. What should be automated before production rollout?

---

# 25. Practice drills

## Easy

Explain the difference between Kubernetes Service and EndpointSlice.

## Medium

Explain CDS, EDS, LDS, RDS, and SDS without reading notes.

## Hard

Explain how one flapping node can increase Istiod CPU and produce customer timeouts in unrelated workloads.

## Bar-raiser

Design configuration scoping for:

- 1,000 services.
- 40,000 pods.
- 12 clusters.
- 3 regions.
- 8 business domains.
- Independent payment and identity trust boundaries.
- Temporary control-plane outage tolerance.
- Selected L7 policy only.

Include:

- Architecture.
- Ownership.
- Migration.
- Failure behavior.
- Metrics.
- Success criteria.
- Rollback.

---

# 26. Personal production-story mapping

Use your own history to build a credible answer.

Write one story with:

## Situation

A registry, configuration, routing, or monitoring system had grown large enough that broad updates or ownership ambiguity caused operational risk.

## Constraint

- Legacy system.
- Many teams.
- Limited maintenance window.
- Global infrastructure.
- Incomplete documentation.
- Customer-facing availability.

## Decision

How did you reduce the failure domain?

Examples:

- Partitioned configuration.
- Introduced ownership.
- Scoped distribution.
- Built an immutable or generated process.
- Added staged rollout.
- Created monitoring and automated rollback.

## Evidence

What data proved the diagnosis?

- CPU.
- Event rate.
- Queue depth.
- Config size.
- Latency.
- Logs.
- Packet capture.
- Database load.
- User error rate.

## Result

Use only metrics you can defend.

## Learning

Tie the story to the mesh scenario:

> The technology was different, but the systems principle was the same: globally distributing irrelevant state turns a local change into a fleet-wide event.

---

# 27. Self-scoring rubric

Score each dimension from 0 to 2.

| Dimension | 0 | 1 | 2 |
|---|---|---|---|
| Problem framing | Says “use Istio” | Mentions scale | Defines graph and failure domains |
| Foundations | Vague DNS answer | Explains proxies | Explains registry, xDS, ACK/NACK |
| Scoping | Only Sidecar | Two mechanisms | Discovery selectors + export + import |
| Failure behavior | Ignores outages | Mentions HA | Explains last-known-good and staleness |
| Operations | Generic monitoring | Names commands | Connects signals to hypotheses |
| Security | Treats visibility as auth | Mentions mTLS | Separates discovery, authn, authz, network |
| Rollout | Big-bang | Canary | Criteria, stop conditions, rollback |
| Trade-offs | One tool is best | Mentions sidecars | Sidecar/Ambient and mesh partition nuance |
| Scale | No model | Mentions config size | Describes churn and fan-out |
| Communication | Tool dump | Structured | Concise opening, deep evidence |

Target:

```text
17/20 or higher
```

---

# 28. Final memorization card

Do not memorize the full chapter. Memorize this chain:

```text
Registry
 -> Istiod watches and computes
 -> xDS distributes
 -> proxies ACK/NACK
 -> broad visibility causes fan-out
 -> discoverySelectors exclude
 -> exportTo limits producer visibility
 -> Sidecar limits consumer import
 -> last-known-good survives temporary control-plane loss
 -> measure churn, config size, push latency, stale proxies
 -> canary and bound the blast radius
 -> Ambient changes dataplane cost, not the need for control-plane discipline
```

Final Principal-level sentence:

> I design service discovery so that endpoint truth remains authoritative, configuration is proportional to real dependencies rather than total mesh size, control-plane churn cannot become a global traffic event, and every proxy has a measurable, bounded, recoverable last-known-good state.

---

# 29. Official primary references

- Istio — Configuration Scoping:  
  https://istio.io/latest/docs/ops/configuration/mesh/configuration-scoping/
- Istio — Debugging Envoy and Istiod:  
  https://istio.io/latest/docs/ops/diagnostic-tools/proxy-cmd/
- Istio — Ambient Mesh documentation:  
  https://istio.io/latest/docs/ambient/
- Istio — Sidecar reference:  
  https://istio.io/latest/docs/reference/config/networking/sidecar/
- Istio — DestinationRule reference:  
  https://istio.io/latest/docs/reference/config/networking/destination-rule/
- Envoy — xDS API overview:  
  https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol
- Kubernetes — Services:  
  https://kubernetes.io/docs/concepts/services-networking/service/
- Kubernetes — EndpointSlices:  
  https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/
