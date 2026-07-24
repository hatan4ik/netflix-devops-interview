# FAANG Principal / Bar-Raiser Architecture Addendum

## Governing standard

Design for partial failure, preserve state and security, bound automation, and validate every architectural claim under stress.

## 1. Istio control-plane scale

The board is right that `Sidecar.egress.hosts` mainly reduces proxy-imported configuration; it does not by itself solve all Istiod ingestion and recomputation costs.

Use three layers together:

1. `discoverySelectors` to ignore non-mesh namespaces early.
2. Producer-side visibility with `networking.istio.io/exportTo` on Kubernetes Services and `spec.exportTo` on Istio resources.
3. Consumer-side imports with `Sidecar.egress.hosts`.

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

Important nuance: Istiod still opens Kubernetes watches broadly; discovery selectors reduce processing by ignoring unselected objects early. `exportTo` reduces visibility and downstream calculation, but it is not identical to eliminating the watch.

Ambient mesh is a strategic option, not a free scalability switch. ztunnel provides shared L4 mTLS, identity, telemetry, and L4 policy. Waypoints provide optional L7 routing, authorization, retries, timeouts, telemetry, and circuit breaking. Ambient reduces per-pod sidecar overhead, but ztunnels and waypoints still consume xDS state and require capacity, policy, observability, rollback, and feature-parity testing.

## 2. eBPF detection versus prevention

The board is right that asynchronous telemetry can be dropped under pressure and must not be represented as an inline security boundary.

Use distinct layers:

- Cilium: networking and identity-aware network policy.
- Hubble: flow visibility.
- Falco or equivalent: behavioral detection.
- Tetragon: in-kernel filtering and optional enforcement.

Tetragon can enforce through return-value override or actions such as `SIGKILL`. However, “SIGKILL always prevents the unauthorized operation before it occurs” is too absolute. Tetragon documents that a signal terminates the process synchronously, but the triggering operation can already have happened for some hooks. Where strict prevention is required, use a supported override/LSM-style hook and combine blocking with termination when appropriate.

Roll out enforcement in monitor mode first, canary by workload or namespace, measure false positives, and maintain an emergency policy-disable path.

## 3. Node fencing, hard replacement, and PDB limits

The board is right that graceful drain can hang when kubelet, runtime, networking, or the API path is degraded. A dead node cannot honor a PDB; PDBs govern voluntary disruption, not infrastructure failure.

Use a state machine:

- Recoverable degradation: cordon, preserve evidence, attempt bounded recovery, then replace if persistent.
- Terminal/poisoned node: fence from traffic and storage, verify replacement capacity, hard replace/terminate, and let controllers recreate workloads.

Do not use Kured as a generic poisoned-node fencing system; it is mainly reboot orchestration. Prefer EKS automatic node repair or a tightly controlled controller with:

- maximum unhealthy-fleet threshold,
- maximum parallel repairs,
- per-condition wait times,
- stateful-workload/storage-fencing rules,
- global kill switch,
- audit trail.

Hard termination is appropriate for terminal failures, but it bypasses graceful shutdown and may expose quorum, volume, duplicate-work, or abrupt-connection risks. The workload architecture must already tolerate involuntary node loss.

## 4. Readiness and cascading failure

The board is right that synchronized deep dependency checks can shrink the ready endpoint pool and concentrate traffic onto survivors.

Readiness should primarily answer whether the local instance can safely provide its minimum serving capability:

- initialization complete,
- listener active,
- credentials loaded,
- internal workers functioning,
- local pools initialized,
- essential local cache/model ready.

The absolute rule “readiness must never consider a dependency” is also too strong. Some services genuinely cannot serve without a mandatory dependency. The safe design avoids uncached, synchronized fleet-wide checks and uses hysteresis, jitter, cached dependency state, degraded modes, and shard-aware health.

Handle downstream slowness with bounded queues, concurrency limits, adaptive load shedding, backpressure, circuit breakers, retry budgets, deadlines, stale responses, and fast `429`/`503` failure. Do not allow retries to multiply across client, gateway, sidecar, and application layers.

## 5. DNS and NodeLocal DNSCache

The board is correct that Istio DNS capture does not cover every DNS path and that high UDP DNS QPS can increase conntrack pressure.

NodeLocal DNSCache provides:

- node-local cache hits,
- reduced cross-node DNS traffic,
- avoidance of kube-proxy DNAT and conntrack for the local path,
- optional TCP upstream queries,
- lower tail latency,
- node-level metrics,
- negative caching.

It is still a per-node dependency. A bad local configuration or stale cache can cause node-wide DNS failure, so roll it out by node pool and monitor agent health, cache behavior, CoreDNS latency, `SERVFAIL`, `NXDOMAIN`, timeouts, TCP fallback, and conntrack utilization.

## 6. Global edge and client-side resilience

The board is right that a single provider is a correlated failure domain and that client-side fallback can bypass some DNS, regional, or edge failures.

Calling AWS Global Accelerator a conventional “single point of failure” is inaccurate: it uses multiple anycast addresses in isolated network zones and health-based endpoint routing. It remains a shared AWS dependency and therefore a correlated-risk domain.

Use layered resilience:

1. Global edge with multiple entry addresses and regional endpoints.
2. Signed, cached client endpoint configuration with bounded regional fallback.
3. Data architecture supporting session reconstruction, idempotency, replication awareness, write fencing, and regional capacity.

Do not race every request across regions. Hedging can double traffic, amplify overload, create duplicate side effects, violate residency rules, and produce inconsistent writes. Limit it to selected idempotent operations with strict budgets and cancellation. Older clients may lack fallback logic, so infrastructure-level failover remains necessary.

## 7. Terraform state decomposition

The board is right that monolithic state couples unrelated lifecycles, permissions, locks, and blast radius.

Split state by:

- blast radius,
- lifecycle,
- ownership,
- privilege boundary,
- deployment frequency,
- recovery semantics.

Typical layers:

```text
organization / identity
network foundation
shared security and observability
regional platform
cluster
data services
application/service tier
```

An application deployment should not lock or require write permission to core-network state.

However, “if `force-unlock` is ever needed, the state is too large” is false. Even a small state can retain a stale lock after a runner crash, backend interruption, or lost network connection. `force-unlock` remains a legitimate exceptional operation after proving there is no active writer.

Avoid over-fragmentation. Excessive micro-states move complexity into CI/CD orchestration, interface versioning, permissions, and cross-state dependency management. Prefer stable contracts through outputs, configuration stores, service catalogs, provider data sources, and explicit pipeline dependencies. Avoid circular remote-state dependencies, and do not use CLI workspaces as a substitute for decomposition requiring separate credentials and controls.

## Final Principal position

The latest review identifies real hyperscale risks, but several claims were too absolute. The final design position is:

1. Scope Istio at control-plane, producer, and consumer layers.
2. Treat Ambient as a different dataplane architecture, not a free optimization.
3. Separate eBPF observation from enforceable kernel controls.
4. Hard-replace terminal nodes only after fencing and within bounded automation.
5. Keep readiness stable and handle dependency pressure with overload controls.
6. Use NodeLocal DNSCache with node-level failure safeguards.
7. Layer global-edge routing with carefully bounded client fallback.
8. Decompose Terraform state by lifecycle and blast radius without creating an unmanageable distributed dependency graph.

> At hyperscale, success means no single component failure, repair action, retry policy, health check, or state lock can expand beyond its intended fault domain.
