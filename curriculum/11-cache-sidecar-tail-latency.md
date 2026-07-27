# Chapter 11 — Cache Sidecar Increasing Tail Latency

## Interview scenario

A sidecar-based caching layer was introduced to reduce origin traffic and improve median latency for a high-volume playback or metadata service. The rollout appears successful: cache hit ratio is high, average latency is lower, and origin QPS has fallen. However, p99 and p99.9 latency now spike periodically, timeout rates increase during traffic bursts and deployments, and only some pods, nodes, request classes, or object sizes are affected.

You are the Staff/Principal SRE leading diagnosis, containment, recovery, and redesign.

---

## 1. What the interviewer is testing

A strong answer must show that you can:

- distinguish lower average latency from better user-visible tail latency;
- prove whether the sidecar is causal rather than merely correlated;
- decompose queueing, cache lookup, serialization, network, and origin latency;
- reason about per-pod cache topology, cold starts, hot keys, and miss amplification;
- debug cgroup CPU throttling, memory pressure, connection pools, event loops, and retries;
- restore service without converting a cache problem into an origin overload;
- design bounded, observable cache behavior around deadlines and service objectives.

The weak answer is: “Increase cache memory” or “add more replicas.”

The Principal answer is:

> First compare identical requests through the cached, bypass, and direct-origin paths in a controlled canary. Then separate hits from misses and decompose the time spent waiting before changing capacity. A cache is successful only when it reduces origin load without creating synchronized queueing, cold-start amplification, or a new shared failure mode.

---

## 2. Clarify what “cache sidecar” means

Two architectures are often described with the same phrase.

### 2.1 Local cache process in every application Pod

```text
client
  |
  v
application container
  |
  +--> localhost / Unix-domain socket
          |
          v
      cache sidecar
          |
          +--> origin on miss
```

Properties:

- very low local hit latency;
- cache lifecycle is coupled to the application Pod;
- every replica has an independent key set and memory budget;
- rolling deployments and HPA scale-outs create cold caches;
- popular objects are duplicated across many Pods;
- load-balancer distribution affects hit ratio;
- application and cache compete for Pod and node resources.

### 2.2 Local caching proxy in front of a shared cache or origin

```text
application
  |
  v
local proxy sidecar
  |
  +--> remote cache cluster
  |
  +--> origin / dependency
```

Properties:

- the sidecar may pool connections, coalesce requests, retry, compress, or transform payloads;
- the remote cache is shared, but every Pod has a separate proxy queue and connection pool;
- proxy limits can produce Pod-local tail spikes even when the remote cache is healthy;
- mesh and cache-proxy retries can multiply each other.

Do not diagnose until the exact request path and ownership of each retry, timeout, pool, and cache entry are known.

---

## 3. Tail latency is not average latency

Let:

- `H` = hit ratio;
- `Lh` = cache-hit latency;
- `Lm` = cache-miss latency including origin work.

A simple average is:

```text
mean latency ≈ H × Lh + (1 − H) × Lm
```

This can improve while p99 gets worse. If misses are rare but extremely slow, they may almost entirely occupy the tail.

Example:

```text
99% of requests: 2 ms hit
 1% of requests: 900 ms miss
```

The average is roughly 11 ms, but the p99 boundary is dominated by the miss path.

### 3.1 Fan-out magnifies tails

If one user request performs 20 independent downstream operations and each operation has a 1% probability of exceeding its own p99 threshold, the probability that at least one operation exceeds that threshold is:

```text
1 − 0.99^20 ≈ 18.2%
```

A locally acceptable tail can therefore become a common end-to-end experience.

### 3.2 Queueing dominates near saturation

When a sidecar event loop, CPU quota, worker pool, lock, or connection pool approaches saturation, wait time rises much faster than service time. p50 may remain healthy while a small group of requests waits behind large objects, slow misses, or throttled workers.

The first question is not “How long does a cache operation take?” It is:

> How long did the request wait before useful cache work began?

---

## 4. Build the end-to-end latency budget

Instrument one request across these stages:

```text
client arrival
  |
  +-- application admission / queue wait
  +-- serialization and sidecar IPC
  +-- sidecar queue wait
  +-- key lookup
  +-- hit response serialization / copy
  |
  +-- on miss:
        +-- request coalescing wait
        +-- upstream pool wait
        +-- DNS / connect / TLS
        +-- remote cache or origin service
        +-- response read and decompression
        +-- cache population
        +-- response copy back to application
```

Each stage needs a bounded deadline and a measurable duration. A single `cache_duration_seconds` metric is insufficient because it hides queueing and miss work inside one number.

---

## 5. Major failure classes

### 5.1 CPU throttling and Pod resource contention

The application and sidecar may be fast individually but unstable together.

Typical causes:

- sidecar CPU limit sized from average rather than burst demand;
- application and sidecar become runnable at the same time and compete for CPU;
- compression, hashing, encryption, or serialization increases sidecar CPU;
- too few event-loop or worker threads for the offered concurrency;
- node CPU pressure adds scheduler delay even without container throttling;
- a mesh proxy adds a third latency-sensitive container to the same Pod.

Evidence:

```bash
kubectl top pod -n playback --containers
kubectl describe pod <pod> -n playback

kubectl exec -n playback <pod> -c cache -- sh -c '
  cat /sys/fs/cgroup/cpu.stat 2>/dev/null || true
  cat /proc/pressure/cpu
  cat /proc/pressure/io
  cat /proc/pressure/memory
'
```

Prometheus signals:

```promql
rate(container_cpu_cfs_throttled_seconds_total{
  namespace="playback", container="cache"
}[5m])
```

```promql
rate(container_cpu_usage_seconds_total{
  namespace="playback", container="cache"
}[5m])
```

A high throttled-time rate correlated with p99 is stronger evidence than CPU utilization alone.

### 5.2 Memory pressure, eviction, allocator, or GC stalls

Typical causes:

- cache memory approaches the container limit;
- eviction work occurs in bursts;
- object churn fragments memory;
- runtime garbage collection pauses request processing;
- `memory.high` reclaim or node pressure introduces stalls before OOM;
- large values create repeated allocation and copy pressure;
- persistence or snapshot work competes for memory bandwidth and I/O.

Inspect:

```bash
kubectl exec -n playback <pod> -c cache -- sh -c '
  cat /sys/fs/cgroup/memory.current 2>/dev/null || true
  cat /sys/fs/cgroup/memory.events 2>/dev/null || true
  cat /proc/pressure/memory
'
```

For Redis-compatible caches, useful commands include:

```bash
redis-cli INFO memory
redis-cli INFO stats
redis-cli INFO commandstats
redis-cli LATENCY LATEST
redis-cli LATENCY DOCTOR
redis-cli SLOWLOG GET 20
```

Do not run high-overhead production diagnostics such as unrestricted command monitoring without understanding the impact.

### 5.3 Cache stampede and miss amplification

A hot key expires or disappears. Thousands of concurrent requests all miss and call the origin.

Origin demand becomes approximately:

```text
origin QPS ≈ request QPS × miss ratio × retry multiplier
```

Without request coalescing, one expired key can turn a local cache event into a fleet-wide dependency outage.

Contributing conditions:

- synchronized TTLs;
- no stale response option;
- no per-key single-flight protection;
- retries at application, cache sidecar, service mesh, and client layers;
- origin timeout longer than the caller’s remaining deadline;
- cache population failure causes every subsequent request to miss again.

### 5.4 Incorrect request coalescing

Single-flight can prevent stampedes, but a bad implementation can serialize unrelated work.

Failure modes:

- one global mutex instead of one key-scoped coordination object;
- hash collision or normalization bug maps unrelated keys together;
- unbounded waiter count;
- the leader request has no deadline;
- leader failure releases all waiters into an origin stampede;
- cancellation of one caller incorrectly cancels shared work;
- large-object population blocks small-object hits.

The correct pattern is key-scoped, deadline-aware, bounded, observable, and failure-safe.

### 5.5 Hot-key imbalance

A small key set receives a disproportionate request volume.

In a per-pod cache, hot keys may be duplicated across every Pod. In a sharded remote cache, one shard may saturate while fleet-wide utilization appears low.

Measure by:

- key popularity rank or sampled heavy hitters;
- hit/miss latency by key class, not raw user identifiers;
- shard, Pod, and node;
- object size;
- command or operation type.

Protect privacy and metric cardinality. Do not export unbounded raw cache keys as labels.

### 5.6 Synchronized TTL expiry

Keys populated during a release, prewarm, batch job, or traffic event may share the same TTL boundary. They later expire together.

Durable controls:

```text
actual TTL = base TTL ± randomized jitter
```

Also use:

- soft TTL before hard TTL;
- stale-while-revalidate;
- background refresh for proven hot keys;
- negative caching for bounded, safe not-found results;
- per-key refresh concurrency limits.

### 5.7 Connection-pool queueing

Cache hits may be local, but misses depend on an upstream pool.

Typical causes:

- pool too small for miss bursts;
- pool too large and overwhelms the origin;
- connection establishment churn;
- per-thread pools cause uneven utilization;
- HTTP/1 requests serialize per connection;
- HTTP/2 stream limits create pending queues;
- idle timeout, max connection age, or rollout causes synchronized reconnects;
- TLS handshakes consume the caller’s latency budget.

For Envoy-based sidecars:

```bash
curl -s localhost:9901/stats | grep -E \
'upstream_rq_pending|upstream_rq_pending_overflow|upstream_cx_connect|upstream_cx_overflow|upstream_rq_timeout|upstream_rq_retry|upstream_rq_time'
```

A larger pool is not automatically safer. The pool must protect the origin as well as reduce local waiting.

### 5.8 Retry amplification

A request may be retried by:

- application cache client;
- sidecar caching proxy;
- service mesh;
- API gateway;
- end client.

Three layers with two attempts each can produce up to eight origin attempts for one user operation.

Use one explicit retry owner per failure class, with:

- a total attempt budget;
- remaining-deadline propagation;
- exponential backoff and jitter;
- retryable-error classification;
- per-origin concurrency limits;
- metrics for original requests versus attempts.

### 5.9 Large-object serialization and copying

The local network hop may be cheap while data movement is expensive.

Potential costs:

- encode/decode;
- compression/decompression;
- user-space copies between application and sidecar;
- kernel socket buffers;
- TLS record processing;
- cache population copy plus response copy;
- garbage creation;
- head-of-line blocking behind a large value.

Always break latency down by object-size bucket. A cache optimized for 4 KiB metadata may behave very differently for multi-megabyte objects.

### 5.10 Cold caches during rollout or autoscaling

This is a defining sidecar-cache risk.

During a Deployment rollout or HPA scale-out:

1. new application Pods start;
2. every new sidecar starts empty;
3. load balancing sends traffic to new Pods;
4. miss ratio rises;
5. origin traffic increases;
6. latency rises and may trigger more scaling;
7. more cold Pods are created.

This feedback loop can make autoscaling reduce effective capacity temporarily.

Mitigations include:

- progressive rollout with origin-load gates;
- minimum warm-up time before full traffic weight;
- controlled prewarming based on a bounded hot-key set;
- consistent request affinity where safe;
- stale data transfer or shared-cache topology when justified;
- scaling on leading demand signals before saturation;
- origin concurrency protection independent of replica count.

Do not mark a Pod ready merely because the cache process is listening. Readiness should reflect whether the application can safely serve its assigned traffic, but it must not wait for the entire cache to become warm.

### 5.11 Per-pod cache fragmentation

With `N` replicas, each sidecar may see only a fraction of requests. If requests are randomly distributed, the effective reuse window per cache shrinks and the aggregate memory footprint grows.

Trade-off:

- more replicas improve compute capacity;
- more independent caches can reduce per-cache hit ratio and increase origin load.

This is one reason cache performance must be evaluated together with HPA behavior and load-balancer policy.

### 5.12 Cache server pauses or blocking operations

For Redis-compatible systems, latency may come from:

- slow or unexpectedly expensive commands;
- persistence or fork-related pauses;
- swapping;
- filesystem or storage latency;
- Transparent Huge Pages;
- failover or resynchronization;
- network interruption;
- CPU overcommit.

Use latency monitoring, slow logs, operating-system evidence, and workload-specific command analysis. Do not assume the cache server is healthy because average command latency is low.

---

## 6. First-response incident strategy: STABILIZE

### Step 1 — Stop expansion

Pause:

- cache-sidecar rollout;
- application rollout that replaces warm Pods;
- HPA scale-down;
- TTL/config changes;
- retry-policy changes;
- mesh or client changes that alter connection churn.

Preserve healthy and unhealthy cohorts for comparison.

### Step 2 — Bound the failure

Create a matrix by:

- application version;
- sidecar version;
- Pod and node;
- region and zone;
- cache hit versus miss;
- key class;
- object-size bucket;
- request class;
- warm versus newly started Pod;
- direct, bypass, and cached path;
- existing versus new upstream connection.

### Step 3 — Protect the origin

Before bypassing the cache broadly, calculate whether the origin can accept the additional QPS.

Use:

- a small bypass canary;
- per-origin concurrency limits;
- rate limits or load shedding;
- stale responses for safe request classes;
- feature degradation;
- traffic shift to healthy regions;
- temporary rollback to the last known-good sidecar.

A fleet-wide bypass can turn a sidecar incident into a dependency collapse.

### Step 4 — Preserve evidence

Collect from one healthy and one unhealthy Pod:

```bash
kubectl get pod <pod> -n playback -o yaml
kubectl describe pod <pod> -n playback
kubectl top pod <pod> -n playback --containers
kubectl logs <pod> -n playback -c cache --since=30m
kubectl logs <pod> -n playback -c app --since=30m
kubectl get events -n playback --sort-by=.lastTimestamp
```

Inside the sidecar:

```bash
ss -s
ss -tin
cat /proc/net/netstat
cat /proc/pressure/cpu
cat /proc/pressure/memory
cat /proc/pressure/io
cat /sys/fs/cgroup/cpu.stat 2>/dev/null || true
cat /sys/fs/cgroup/memory.events 2>/dev/null || true
```

Capture profiles only with a bounded method appropriate to the runtime and incident risk.

---

## 7. Prove causality with a controlled experiment

Use the same request corpus and compare:

| Path | Purpose | Required guardrail |
|---|---|---|
| Normal sidecar cache | Baseline failing behavior | Existing production limits |
| Cache bypass through same sidecar | Separates lookup/cache logic from proxy/network logic | Small canary; origin concurrency cap |
| Direct dependency path | Separates entire sidecar from origin behavior | Strong authentication and tiny traffic slice |
| Known-hit replay | Tests local hit path | Fixed safe keys and object sizes |
| Forced-miss replay | Tests coalescing and origin path | Strict rate limit |
| Warm Pod vs new Pod | Tests cold-start effect | Same application and sidecar versions |

A useful result looks like:

```text
known hit through sidecar: 2 ms
cache bypass through sidecar: 180 ms
controlled direct origin: 35 ms
```

That points toward sidecar proxy queueing, pool behavior, or retries—not the origin itself.

Change one variable at a time. Do not compare different regions, object sizes, versions, and traffic levels and call the result causal.

---

## 8. Observability that exposes the mechanism

### 8.1 Histograms

Record histograms for:

- total request duration;
- application-to-sidecar IPC;
- sidecar queue wait;
- cache lookup duration;
- request-coalescing wait;
- upstream pool wait;
- connect and TLS time;
- origin duration;
- serialization and response-copy time.

Useful bounded dimensions:

- `result="hit|miss|stale|bypass|error"`;
- request class;
- object-size bucket;
- sidecar version;
- region and zone;
- warm/cold cohort;
- upstream cluster;
- retry attempt.

Avoid cache key, user ID, title ID, or other unbounded labels.

Example p99 query with a classic histogram:

```promql
histogram_quantile(
  0.99,
  sum by (le, result) (
    rate(cache_sidecar_request_duration_seconds_bucket[5m])
  )
)
```

Use bucket boundaries that resolve the actual SLO, not generic defaults.

### 8.2 Counters and gauges

Track:

- hits, misses, stale serves, bypasses, and population failures;
- unique origin requests versus total origin attempts;
- coalesced leaders and waiters;
- rejected or timed-out waiters;
- queue depth;
- pool active, idle, pending, and overflow;
- current entries and bytes;
- evictions and expirations;
- GC pause and event-loop delay;
- CPU throttled time;
- cold-Pod count and warm-up age;
- origin QPS and saturation.

### 8.3 Tracing

A trace should answer:

- Did the application wait before calling the sidecar?
- Was the lookup a hit, miss, stale serve, or bypass?
- Did the request wait behind a single-flight leader?
- How long did it wait for an upstream connection?
- How many attempts occurred?
- Which deadline remained at every hop?
- Did cache population complete after the caller timed out?

Sample errors and slow requests at a higher rate, but preserve enough unbiased sampling to estimate population behavior.

---

## 9. Safe mitigation hierarchy

Prefer the smallest reversible mitigation that restores the SLO without overloading the origin.

1. **Pause the rollout and preserve warm healthy Pods.**
2. **Route away from the bad sidecar revision or affected node cohort.**
3. **Disable an expensive cache feature** such as compression, persistence, or a problematic request class.
4. **Serve bounded stale data** where product semantics allow it.
5. **Reduce concurrency or shed low-priority work** before queues grow without bound.
6. **Correct CPU/memory headroom** when evidence proves throttling or reclaim.
7. **Adjust pool and circuit-breaker limits together** with origin capacity.
8. **Add TTL jitter or temporarily extend safe TTLs** to stop synchronized refresh.
9. **Bypass a small request class or traffic percentage** with an origin budget.
10. **Roll back the sidecar** if the new version is causal.

Avoid:

- fleet-wide cache flush;
- fleet-wide bypass;
- mass Pod restart;
- unbounded connection-pool increase;
- retries without a total budget;
- raising memory limits without understanding eviction, fragmentation, and node capacity;
- reducing TTLs during an origin incident;
- treating hit ratio as proof of health.

---

## 10. Production design patterns

### 10.1 Soft TTL, hard TTL, and stale-while-revalidate

```text
fresh until soft TTL
    |
    +--> one bounded refresher starts
    +--> other callers receive stale-but-acceptable value

invalid after hard TTL
    |
    +--> fail, degrade, or call origin according to policy
```

This prevents every caller from blocking on refresh.

### 10.2 Bounded key-scoped coalescing

For each key:

- allow one leader;
- cap waiter count;
- propagate deadlines without letting one short caller cancel shared work;
- release waiters safely on leader failure;
- prevent immediate stampede after failure;
- record wait duration and outcome;
- garbage-collect coordination state.

### 10.3 Deadline budgeting

Example for a 300 ms request budget:

```text
application admission        20 ms
cache lookup                 10 ms
coalescing / pool wait       30 ms
connect / TLS                30 ms
origin                      160 ms
serialization                20 ms
network and safety reserve   30 ms
```

No downstream timeout should exceed the caller’s remaining deadline.

### 10.4 Admission control

A bounded queue turns overload into explicit rejection instead of unpredictable multi-second latency.

Choose overload behavior by request class:

- serve stale metadata;
- return a degraded response;
- drop speculative prefetch;
- reject low-priority refresh;
- preserve entitlement, playback-start, or other critical paths;
- prevent background work from consuming foreground capacity.

### 10.5 Resource isolation

Kubernetes schedules a Pod based on declared requests, and the Pod’s regular-container resources contribute to its total request. Give both application and sidecar explicit requests based on measured burst behavior.

Example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: playback-metadata
spec:
  replicas: 20
  strategy:
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 2
  template:
    metadata:
      labels:
        app: playback-metadata
    spec:
      terminationGracePeriodSeconds: 45
      containers:
        - name: app
          image: example/playback-app@sha256:APP_DIGEST
          resources:
            requests:
              cpu: "750m"
              memory: "768Mi"
            limits:
              memory: "1Gi"
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            periodSeconds: 5
            timeoutSeconds: 1
            failureThreshold: 2
        - name: cache
          image: example/cache-sidecar@sha256:CACHE_DIGEST
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              memory: "768Mi"
          ports:
            - name: cache
              containerPort: 7379
          readinessProbe:
            httpGet:
              path: /ready
              port: 9090
            periodSeconds: 5
            timeoutSeconds: 1
            failureThreshold: 2
```

This is an illustrative starting point, not a universal sizing recommendation. Validate whether CPU limits are appropriate for the latency-sensitive runtime; a poorly chosen quota can create throttling. Memory still needs a limit and a safe eviction policy.

### 10.6 Lifecycle and draining

On startup:

1. sidecar process becomes locally responsive;
2. configuration and credentials load;
3. upstream connectivity is validated with a bounded check;
4. application becomes ready;
5. traffic weight grows progressively;
6. cache warmth and origin load are monitored.

On termination:

1. remove the Pod from new traffic;
2. stop accepting new refresh leaders;
3. complete or cancel bounded in-flight work;
4. flush only required state;
5. close upstream connections;
6. terminate within the grace period.

When native Kubernetes sidecar containers are used, verify startup and shutdown ordering explicitly. Do not assume an arbitrary regular sidecar starts before the application.

### 10.7 Topology decision

| Topology | Advantages | Risks |
|---|---|---|
| In-process cache | Lowest call overhead | Language-specific, application memory coupling |
| Per-Pod sidecar cache | Language-neutral, local latency, Pod-level isolation | Cold starts, duplicated memory, fragmented hit rate, one-to-one scaling |
| Node-local cache | Better sharing than per-Pod | Noisy-neighbor and node-failure domain |
| Shared remote cache | Shared working set and independent scaling | Network latency and larger shared blast radius |
| Multi-tier cache | Local speed plus shared reuse | More invalidation, consistency, and observability complexity |

Choose based on working-set size, consistency, deployment frequency, origin capacity, fault isolation, and operational ownership—not fashion.

---

## 11. Rollout gates

A sidecar-cache release should progress through:

```text
offline benchmark
  -> integration test
  -> representative load test
  -> one-Pod canary
  -> one-zone canary
  -> small regional percentage
  -> staged fleet rollout
```

Gate on:

- p99 and p99.9 by hit and miss;
- origin QPS and saturation;
- cold-Pod miss ratio;
- CPU throttling and PSI;
- queue and pool pending depth;
- retry amplification;
- eviction and GC/fork pauses;
- error and timeout rate;
- playback-start or business SLO;
- rollback time.

Test these transitions separately:

- sidecar binary upgrade;
- cache schema or key format change;
- TTL policy change;
- application rollout;
- mesh rollout;
- node-image rollout;
- HPA scale-out;
- regional failover.

Do not combine them into one unobservable release.

---

## 12. Hands-on failure-injection lab

### Lab objective

Demonstrate that median latency can improve while p99 worsens, then identify and remove the queueing mechanism.

### Suggested components

- HTTP application;
- local cache sidecar or caching proxy;
- controllable origin service;
- Prometheus-compatible metrics;
- distributed tracing;
- load generator with hot-key and object-size distributions.

### Experiments

1. **Synchronized expiry**
   - Populate 1,000 keys with the same TTL.
   - Drive steady traffic.
   - Observe the expiry boundary, miss burst, and origin QPS.
   - Add TTL jitter and compare.

2. **CPU throttling**
   - Set an intentionally low sidecar CPU limit.
   - Increase concurrency while preserving request mix.
   - Correlate `cpu.stat`, throttled time, queue wait, and p99.

3. **Global lock bug**
   - Serialize all misses behind one lock.
   - Compare unrelated-key latency.
   - Replace with key-scoped bounded coalescing.

4. **Cold scale-out**
   - Scale from 5 to 25 Pods.
   - Measure hit ratio, origin QPS, and time to warm.
   - Add progressive traffic weighting or bounded prewarm.

5. **Retry amplification**
   - Introduce 300 ms origin latency and a retryable error.
   - Enable retries at two layers.
   - Compare user requests with origin attempts.
   - designate one retry owner and retest.

6. **Large-object head-of-line blocking**
   - Mix 4 KiB and 4 MiB objects.
   - Measure queue wait and copy/serialization time by size bucket.
   - isolate large objects or apply concurrency classes.

### Exit criteria

The learner must produce:

- a request-path diagram;
- a causal experiment showing the sidecar’s contribution;
- p50/p95/p99/p99.9 histograms by hit/miss and object size;
- an origin-protection calculation;
- a reversible mitigation;
- a durable design change;
- rollout and rollback gates.

---

## 13. Principal-level 90-second answer

> I would first pause the sidecar rollout and preserve a healthy and unhealthy cohort. I would not bypass the cache fleet-wide because that could overload the origin. In a controlled canary, I would replay the same requests through the normal cache path, cache bypass through the sidecar, and direct origin path. I would split hits from misses and trace application queueing, sidecar queue wait, lookup, coalescing, upstream pool wait, origin time, and serialization.
>
> I would correlate p99 with sidecar CPU throttling, PSI, event-loop delay, memory and eviction, object size, pool pending depth, retries, TTL boundaries, hot keys, and Pod age. I would specifically test for cold-cache amplification during rollout or HPA scale-out, because adding Pods can temporarily increase origin traffic and worsen effective capacity.
>
> The immediate mitigation is the smallest reversible action: route away from the bad revision, preserve warm Pods, serve bounded stale data, shed low-priority refreshes, or bypass only a protected traffic slice. The durable fix is bounded key-scoped coalescing, TTL jitter, stale-while-revalidate, one retry owner, deadline propagation, explicit resource headroom, origin concurrency protection, and rollout gates on p99 and origin load—not hit ratio alone.

---

## 14. Likely follow-up questions

### Why can p50 improve while p99 worsens?

Hits become very fast, but rare misses, queueing, throttling, or synchronized refresh dominate the tail.

### Why not immediately disable the cache?

The origin may not have capacity for full request volume. A fleet-wide bypass can create a larger outage.

### Why can adding Pods make the problem worse?

Each new sidecar may start cold. More Pods can fragment the working set, lower per-cache reuse, and increase aggregate miss traffic.

### How do you distinguish origin latency from sidecar queueing?

Compare a cache bypass through the sidecar with a controlled direct-origin path, and instrument pool wait separately from origin service time.

### How do you prevent stampedes?

Use TTL jitter, soft/hard TTLs, stale-while-revalidate, bounded key-scoped single-flight, negative caching where safe, and origin concurrency limits.

### Should the cache sidecar have a CPU limit?

There is no universal answer. A strict quota can create throttling and tail latency; no limit can allow noisy-neighbor behavior. Measure burst demand, node isolation, and policy requirements, then validate under realistic p99 load.

### When would you replace the sidecar with a shared cache?

When duplicated memory, cold-start behavior, fragmented hit ratio, or one-to-one scaling outweigh the local-latency and isolation benefits. The shared cache must still be partitioned and protected to avoid a larger blast radius.

### What metric would you never trust alone?

Fleet-wide cache hit ratio. It can hide one bad region, hot-key misses, cold Pods, retry amplification, and a severely degraded p99.

---

## 15. Review checklist

Before declaring the incident resolved, confirm:

- [ ] cached, bypass, and direct paths were compared safely;
- [ ] hit and miss latency are separated;
- [ ] queue wait is measured separately from service time;
- [ ] origin QPS and attempts are within budget;
- [ ] retries have one owner and a total budget;
- [ ] TTLs are jittered where appropriate;
- [ ] coalescing is key-scoped and bounded;
- [ ] CPU throttling, PSI, memory pressure, eviction, and GC are understood;
- [ ] object-size distribution is represented;
- [ ] cold-Pod behavior was tested;
- [ ] rollout and HPA interactions were tested;
- [ ] stale and degradation semantics are product-approved;
- [ ] the rollback path is proven;
- [ ] SLO dashboards expose p99/p99.9 and origin protection.

---

## 16. References

- Kubernetes: Resource Management for Pods and Containers — https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
- Kubernetes: Sidecar Containers — https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/
- Redis: Diagnosing Latency Issues — https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency/
- Redis: Latency Monitoring — https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency-monitor/
- Envoy: Connection Pooling — https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/connection_pooling
- Envoy: Statistics Overview — https://www.envoyproxy.io/docs/envoy/latest/operations/stats_overview.html
- Prometheus: Histograms and Summaries — https://prometheus.io/docs/practices/histograms/

---

## Core principle

> A cache is not healthy because its hit ratio is high or its average latency is low. It is healthy when hit, miss, refresh, retry, rollout, and failure behavior remain bounded under the service’s real traffic distribution—and when the origin stays protected while the user-visible tail meets its objective.
