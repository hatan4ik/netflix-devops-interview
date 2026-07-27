# Chapter 12 — Requests Returning 504 While Health Checks Remain Green

## Interview scenario

A playback, manifest, or API path begins returning HTTP `504 Gateway Timeout`. Cloud load balancer targets remain healthy, Kubernetes Pods are Ready, sidecar health endpoints return `200`, and dashboards show no obvious fleet-wide CPU or memory crisis. The problem affects only selected regions, request classes, devices, content sizes, or newly deployed cohorts.

You are the Staff/Principal SRE leading diagnosis, containment, recovery, and prevention.

---

## 1. What the interviewer is testing

A strong answer must show that you can:

- identify which hop generated the `504` before guessing at root cause;
- distinguish health-check success from real request-path success;
- trace one request across CDN, load balancer, gateway, mesh, application, dependency, and storage layers;
- reason about connect, request, per-try, idle, stream, and application deadlines;
- separate service time from queue wait, connection-pool wait, retries, and network loss;
- restore service without hiding overload behind longer timeouts;
- redesign health, deadlines, retries, capacity, and observability around user-visible objectives.

The weak answer is: “Increase the load balancer timeout.”

The Principal answer is:

> A `504` tells me which intermediary stopped waiting; it does not tell me which upstream component caused the delay. I first identify the issuer, trace one failed request end to end, find the first exhausted deadline, and compare the real request path with the health-check path. Then I route around or degrade the smallest failing dependency while protecting the rest of the system from retry and queue amplification.

---

## 2. What a 504 does—and does not—mean

A gateway or proxy returns `504` when it cannot obtain a complete upstream response within the deadline or idle-time rules it enforces.

The status code does **not** prove that:

- the gateway itself is overloaded;
- the immediately adjacent upstream is the root cause;
- the application process is dead;
- Kubernetes readiness is wrong;
- increasing a timeout will restore safe capacity.

Possible issuers include:

- CDN or global edge;
- cloud application load balancer;
- API gateway or ingress controller;
- Envoy edge gateway;
- service-mesh sidecar;
- application gateway library;
- an upstream service acting as a gateway to another dependency.

The first incident question is therefore:

> Which component created this exact response?

Use access logs, response headers, request IDs, proxy response flags, response-code details, and cloud metrics to answer that question.

---

## 3. Why green health checks can coexist with broken traffic

A health check is only evidence for the exact thing it tested.

Health traffic often differs from production traffic in:

- path;
- host or `:authority` header;
- port;
- protocol version;
- TLS/SNI behavior;
- authentication and authorization;
- payload and response size;
- byte-range behavior;
- dependency fan-out;
- cache hit ratio;
- concurrency;
- geographic origin;
- connection reuse;
- request timeout;
- routing rule or target subset.

Examples:

- `/healthz` returns from memory while `/playback/start` calls entitlement, DRM, metadata, personalization, and origin-selection services.
- The load balancer checks port `8080`, while user traffic reaches an Envoy listener on `8443`.
- Health checks use HTTP/1.1, while user traffic uses HTTP/2 or gRPC.
- A health check returns 200 before a response body is streamed, while large segment requests stall after headers.
- A readiness endpoint tests local event-loop responsiveness but not an overloaded database connection pool.
- The health route hits a default virtual host, while real requests match a broken host-based route.
- Health checks are tiny and infrequent; real traffic exposes packet loss, CPU throttling, pool saturation, or head-of-line blocking.

Kubernetes readiness controls whether a Pod receives Service traffic, but readiness success does not prove that every business request and dependency chain can complete within its SLO.

---

## 4. Build the real end-to-end path

For a streaming request, draw the actual path before changing configuration:

```text
client / device
  |
  v
ISP / internet
  |
  v
CDN or global edge
  |
  v
cloud load balancer
  |
  v
ingress / Envoy gateway
  |
  v
service-mesh sidecar
  |
  v
playback or manifest service
  |
  +--> identity / authentication
  +--> entitlement
  +--> DRM or license service
  +--> title metadata
  +--> personalization / experimentation
  +--> origin selection
  |
  v
cache / object storage / segment origin
  |
  v
response streamed back through every timeout boundary
```

For each hop, record:

- owner;
- timeout and idle-timeout values;
- retry owner and maximum attempts;
- connection-pool limit;
- queue limit;
- concurrency limit;
- health signal;
- access-log location;
- request-ID propagation;
- fallback behavior;
- capacity and saturation indicators.

---

## 5. Timeout taxonomy

“Timeout” is not one setting.

### 5.1 Connect timeout

Maximum time to establish the upstream TCP connection.

Possible root causes:

- no listener;
- security group, NACL, NetworkPolicy, or routing issue;
- SYN loss or retransmission;
- connection backlog saturation;
- stale endpoint;
- ephemeral-port or conntrack pressure;
- target too overloaded to accept promptly.

### 5.2 TLS handshake timeout

Maximum time to complete the TLS negotiation after TCP connection.

Possible root causes:

- certificate or trust mismatch;
- overloaded proxy or server;
- packet loss;
- stalled cryptographic processing;
- broken SNI or protocol path;
- mTLS policy inconsistency.

Some intermediaries expose this as a connection or gateway timeout rather than a clear certificate error.

### 5.3 Request or route timeout

Maximum total time allowed for an upstream response, usually including some or all retry attempts.

A route timeout can expire while the application is still computing or waiting on a dependency.

### 5.4 Per-try timeout

Maximum time for one upstream attempt when retries are enabled.

It must fit inside the total request deadline with room for backoff, connection setup, and response transmission.

### 5.5 Stream idle timeout

Maximum time with no meaningful stream activity.

This matters for:

- large downloads;
- server-sent events;
- gRPC streams;
- chunked responses;
- slow object storage reads;
- connections where headers arrive but body bytes stall.

A stream can exceed a long wall-clock duration safely if bytes continue flowing, yet fail a short idle timeout when no data moves.

### 5.6 Connection idle timeout

Maximum idle time for a pooled downstream or upstream connection between requests.

Mismatched keepalive and idle timeouts can cause resets and retries around reused connections. These often appear as `502`, but retry behavior can later consume the total budget and produce `504`.

### 5.7 Maximum stream duration

An absolute lifetime limit for a stream, independent of whether bytes continue moving.

### 5.8 Application and dependency deadlines

Examples:

- HTTP client timeout;
- database statement timeout;
- lock-acquisition timeout;
- worker-queue timeout;
- object-store SDK timeout;
- gRPC deadline;
- asynchronous task wait timeout.

An application may remain “healthy” while every request waits behind one of these boundaries.

---

## 6. Design the timeout hierarchy

For a synchronous request path, inner work should stop before the outer caller gives up.

Conceptually:

```text
client deadline
  > CDN / edge deadline
    > gateway route deadline
      > service request deadline
        > dependency deadline
          > individual attempt timeout
```

Example for a 2,000 ms user-visible budget:

```text
client deadline                    2,000 ms
edge / gateway budget              1,800 ms
service budget                     1,500 ms
dependency total budget              900 ms
single attempt                       350 ms
backoff + second attempt             450 ms
response serialization / reserve     600 ms
```

This is illustrative, not a universal configuration.

### 6.1 Propagate remaining deadline

Every service should know the caller’s remaining budget through an explicit deadline mechanism, such as a gRPC deadline or a validated internal request-deadline header.

Do not create a fresh full timeout at every hop. That allows abandoned work to continue long after the user has received a `504`.

### 6.2 Preserve cancellation

When the caller times out or disconnects:

- cancel downstream work where safe;
- release database connections and locks;
- stop unnecessary object reads;
- avoid populating expensive responses nobody will consume;
- record cancellation separately from server error.

### 6.3 Do not “solve” overload by extending all timeouts

Longer timeouts retain more requests, sockets, memory, worker slots, and locks. Under saturation, this can increase concurrency and make the queue collapse worse.

Increase a timeout only when:

- the operation is legitimately expected to take longer;
- the user or protocol can tolerate it;
- downstream capacity and cancellation are safe;
- the new value preserves the deadline hierarchy;
- load tests prove the tail remains bounded.

---

## 7. First-response incident strategy: STABILIZE

### Step 1 — Stop expansion

Pause changes that may erase evidence or increase load:

- gateway, sidecar, and application rollouts;
- timeout changes;
- retry-policy changes;
- HPA scale-down;
- node replacement affecting only one cohort;
- connection-pool or circuit-breaker changes;
- health-check changes;
- broad cache bypass or flush.

### Step 2 — Identify the 504 issuer

At every proxy layer, inspect:

- status code;
- response flags;
- response-code details;
- total duration;
- upstream service time;
- upstream host and cluster;
- bytes received and sent;
- retry count;
- connection and transport failure details;
- request ID and trace ID.

For Envoy, add structured access-log fields such as:

```text
%RESPONSE_CODE%
%RESPONSE_FLAGS%
%RESPONSE_CODE_DETAILS%
%DURATION%
%REQUEST_DURATION%
%RESPONSE_DURATION%
%RESPONSE_TX_DURATION%
%UPSTREAM_HOST%
%UPSTREAM_CLUSTER%
%UPSTREAM_TRANSPORT_FAILURE_REASON%
%UPSTREAM_CONNECTION_ID%
%UPSTREAM_REQUEST_ATTEMPT_COUNT%
%BYTES_RECEIVED%
%BYTES_SENT%
%REQUEST_HEADER(X-REQUEST-ID)%
```

Field availability depends on Envoy version and context. Validate the exact formatter names in the deployed version.

Useful Envoy clues include:

- `UT` / upstream request timeout;
- `SI` / stream idle timeout;
- `UF` / upstream connection failure;
- `UO` / upstream overflow;
- `URX` / upstream retry limit exceeded;
- `UH` / no healthy upstream;
- `UC` / upstream connection termination;
- response-code details such as `upstream_response_timeout`.

A response flag narrows the failure layer but does not always identify the systemic cause.

### Step 3 — Trace one failed request

Choose one representative failure and follow the same request or trace ID across:

1. CDN or edge log;
2. cloud load-balancer access log;
3. ingress or gateway log;
4. source sidecar log;
5. application log and trace;
6. destination sidecar or service log;
7. database, cache, object store, or downstream dependency;
8. response path.

The objective is to find the first component that stopped making progress—not merely the final component that emitted `504`.

### Step 4 — Build the failure matrix

Compare by:

- region and availability zone;
- gateway and sidecar version;
- application version;
- source and destination Pod;
- node pool and instance type;
- request path;
- device, codec, bitrate, or protocol;
- object-size and byte-range bucket;
- cache hit versus miss;
- new versus reused connection;
- HTTP/1.1 versus HTTP/2 or gRPC;
- health path versus business path;
- normal versus high concurrency;
- newly started versus long-running Pod.

### Step 5 — Protect dependencies

Before scaling callers or enabling retries, inspect the downstream’s capacity.

Use:

- bounded concurrency;
- admission control;
- rate limiting;
- load shedding;
- stale or cached fallback;
- traffic shifting;
- feature degradation;
- caller-side retry reduction.

Scaling the caller can increase pressure on the actual bottleneck.

---

## 8. Major failure classes

### 8.1 Queueing with low CPU

Low CPU does not prove spare capacity.

Requests may wait for:

- database connections;
- worker threads;
- async executor slots;
- upstream connections;
- locks;
- rate-limit tokens;
- file descriptors;
- object-store SDK permits;
- per-tenant or per-key serialization.

Measure queue wait separately from service time.

### 8.2 Connection-pool exhaustion

A proxy or application can have healthy endpoints but no immediately available upstream connection.

Envoy-oriented evidence:

```bash
curl -s localhost:15000/stats | grep -E \
'upstream_rq_pending|upstream_rq_pending_overflow|upstream_cx_connect_timeout|upstream_cx_overflow|upstream_rq_timeout|upstream_rq_retry'
```

Depending on the deployment, the admin port may differ.

A larger pool can reduce local wait but overload the destination. Tune pool limits together with destination concurrency and queue limits.

### 8.3 Retry amplification

If client, gateway, mesh, and application each retry, one user request can produce many dependency attempts.

For `n` layers with `a` maximum attempts each:

```text
worst-case attempts = a^n
```

Three layers with two attempts each can create up to eight attempts.

Control retries with:

- one owner per failure class;
- total attempt budget;
- remaining-deadline propagation;
- exponential backoff and jitter;
- retryable-error classification;
- retry concurrency limits;
- request-versus-attempt metrics.

### 8.4 Health check bypasses the failing dependency

A local `/ready` endpoint can remain green while:

- entitlement is unavailable;
- DRM is saturated;
- object storage is slow;
- metadata queries are blocked;
- a required certificate is stale;
- one route has no functioning upstream pool.

Do not put every remote dependency into liveness. Use shallow liveness, carefully designed readiness, capability health, and external business synthetics.

### 8.5 Large-response and streaming failures

Small requests pass; large manifests or segment responses fail.

Investigate:

- stream idle timeout;
- packet loss and retransmission;
- MTU or fragmentation issues;
- HTTP/2 flow-control windows;
- response buffering;
- compression;
- slow object-store reads;
- byte-range handling;
- incorrect `Content-Length`;
- application writes that stall after headers;
- client or gateway backpressure.

Cloud load balancers may time out while waiting for missing body bytes when a target advertises a larger `Content-Length` than it sends.

### 8.6 DNS resolves to a reachable but wrong endpoint

DNS can succeed while routing traffic to:

- a stale address;
- an old region;
- an under-capacity endpoint;
- a public endpoint instead of private connectivity;
- the wrong service version;
- a healthy IP serving the wrong virtual host.

Compare DNS answers, endpoint ownership, SNI, route selection, and actual upstream host from the proxy log.

### 8.7 Network loss invisible to health checks

Tiny periodic health requests may survive while large or bursty traffic suffers.

Inspect:

- TCP retransmissions;
- SYN retransmissions;
- receive/transmit drops;
- MTU and path MTU discovery;
- conntrack pressure;
- ephemeral-port pressure;
- NACL ephemeral-port rules;
- CNI or eBPF drops;
- cross-zone network asymmetry;
- NIC queue and node pressure.

Node commands:

```bash
ss -s
ss -tin
nstat -az | grep -E 'Retrans|Timeout|Listen|Abort'
cat /proc/net/netstat
cat /proc/pressure/cpu
cat /proc/pressure/memory
cat /proc/pressure/io
conntrack -S 2>/dev/null || true
```

### 8.8 Database or storage lock contention

The process and Pod remain healthy while requests wait on:

- row or table locks;
- connection-pool acquisition;
- long-running queries;
- checkpoint or compaction activity;
- storage throttling;
- object-store hot partitions;
- metadata service serialization.

Correlate the request trace with database wait events, query duration, lock ownership, pool pending depth, and storage latency.

### 8.9 Event-loop or worker starvation

An application can accept health requests on one thread or priority path while business work is starved.

Causes include:

- blocking I/O on an event loop;
- unbounded callback work;
- thread-pool exhaustion;
- runtime GC pauses;
- CPU throttling;
- priority inversion;
- expensive logging or serialization;
- runaway background tasks.

### 8.10 Rollout and drain mismatch

During rollout:

- a Pod may remain in a target group after it stopped accepting work;
- readiness may turn false after the load balancer has already sent requests;
- termination grace may be shorter than request duration;
- keepalive connections may continue carrying traffic to a draining target;
- gateway and application versions may disagree on timeout or protocol behavior.

Validate:

- readiness removal;
- target deregistration delay;
- `preStop` timing;
- connection draining;
- graceful shutdown;
- maximum request duration;
- termination grace;
- in-flight request tracking.

### 8.11 Hidden route or subset asymmetry

Only one host, path, header, tenant, codec, or device class may route to a different cluster or subset.

Inspect actual proxy routes, clusters, and endpoints—not only Git intent.

```bash
istioctl proxy-config routes <gateway-pod> -n <namespace>
istioctl proxy-config clusters <gateway-pod> -n <namespace>
istioctl proxy-config endpoints <gateway-pod> -n <namespace>
istioctl proxy-status
```

### 8.12 Capacity exists but cannot be used

Examples:

- HPA created Pods that are Pending;
- Pods are Running but not Ready;
- EndpointSlices are stale;
- load balancer target registration lags;
- one zone has no healthy capacity;
- topology policy keeps traffic local to a saturated zone;
- new Pods are cold and overload dependencies;
- subnet IP exhaustion blocks scale-out.

Follow the capacity chain end to end rather than stopping at desired replicas.

---

## 9. Cloud load-balancer investigation

For an AWS Application Load Balancer, distinguish load-balancer-generated errors from target-generated errors using access logs and CloudWatch metrics.

Useful signals include:

- `HTTPCode_ELB_5XX_Count`;
- `HTTPCode_Target_5XX_Count`;
- `TargetResponseTime`;
- `HealthyHostCount` and `UnHealthyHostCount`;
- target connection errors;
- rejected connections;
- access-log fields showing target status code and processing times.

Commands:

```bash
aws elbv2 describe-target-health \
  --target-group-arn <target-group-arn>

aws elbv2 describe-load-balancer-attributes \
  --load-balancer-arn <load-balancer-arn>

aws elbv2 describe-target-group-attributes \
  --target-group-arn <target-group-arn>
```

Common ALB-generated `504` causes include:

- target connection timeout;
- target does not respond before idle timeout;
- TLS handshake timeout to the target;
- NACL blocks return traffic on ephemeral ports;
- malformed response length where advertised bytes never arrive.

Do not assume every cloud or environment uses the same timeout defaults. Inspect the deployed attributes.

---

## 10. Kubernetes and mesh investigation

### 10.1 Kubernetes state

```bash
kubectl get deploy,rs,pod,svc,endpointslice -n <namespace> -o wide
kubectl get pod <pod> -n <namespace> -o yaml
kubectl describe pod <pod> -n <namespace>
kubectl get events -n <namespace> --sort-by=.lastTimestamp
kubectl top pod -n <namespace> --containers
kubectl top node
```

Check:

- readiness transitions;
- container restarts;
- CPU throttling and memory pressure;
- endpoint membership;
- node and zone placement;
- rollout cohorts;
- Pending capacity;
- Pod age;
- termination state.

### 10.2 Envoy data plane

```bash
istioctl proxy-status
istioctl analyze -A
istioctl proxy-config listeners <pod> -n <namespace>
istioctl proxy-config routes <pod> -n <namespace>
istioctl proxy-config clusters <pod> -n <namespace>
istioctl proxy-config endpoints <pod> -n <namespace>
```

From the Envoy admin interface, when safely exposed only to the local Pod:

```bash
curl -s localhost:15000/stats
curl -s localhost:15000/clusters
curl -s localhost:15000/config_dump
```

Compare a healthy and unhealthy proxy snapshot.

### 10.3 Test each boundary

Use the same host, headers, authentication, payload, and deadline where possible.

```bash
curl -sS -o /dev/null \
  -w 'code=%{http_code} dns=%{time_namelookup} connect=%{time_connect} tls=%{time_appconnect} starttransfer=%{time_starttransfer} total=%{time_total}\n' \
  https://service.example.com/real-business-path
```

Controlled tests may compare:

- external edge path;
- load balancer path;
- gateway Service;
- direct gateway Pod;
- application Service;
- direct application Pod;
- direct dependency.

Do not bypass security or expose internal services. Use approved debug Pods, identities, and tiny traffic volumes.

---

## 11. Observability contract

### 11.1 Structured access logs

At every proxy, capture:

- timestamp;
- request and trace IDs;
- downstream address and protocol;
- route and virtual host;
- response code;
- response flags and details;
- duration;
- time to first upstream byte;
- upstream host and cluster;
- retry attempt count;
- bytes received and sent;
- termination reason;
- deployment version, region, and zone.

### 11.2 Distributed tracing

The trace must expose:

- queue wait;
- connection-pool wait;
- DNS, connect, and TLS time;
- dependency duration;
- retries as separate attempts;
- remaining deadline;
- cancellation;
- response serialization and streaming duration.

A trace with one giant “HTTP client” span is insufficient.

### 11.3 Metrics

Measure:

- 504 rate by issuer;
- success and latency by business path;
- p50/p95/p99/p99.9;
- route and dependency timeout counts;
- pool pending and overflow;
- queue wait;
- original requests versus attempts;
- connection establishment and TLS duration;
- retransmits and resets;
- request and response-size buckets;
- byte-range success;
- target and endpoint health;
- dependency saturation;
- cold-Pod and rollout cohort.

Avoid unbounded labels such as raw user IDs, title IDs, URLs, or request IDs.

---

## 12. Health model that matches user reality

Use several distinct layers.

### 12.1 Liveness

Answers:

> Is the local process irrecoverably stuck, and is restart likely to cure it?

Keep it shallow and local. Do not restart every Pod because a shared database is slow.

### 12.2 Readiness

Answers:

> Should this Pod receive new traffic now?

It may include bounded local capability checks and carefully cached critical dependency state, but it must not perform expensive fan-out on every probe.

### 12.3 Capability health

Expose whether specific functions are currently available, such as:

- personalized manifests;
- entitlement checks;
- DRM license acquisition;
- origin selection;
- large-object streaming;
- write operations.

This allows deliberate degradation rather than binary “healthy/unhealthy.”

### 12.4 External synthetic transaction

A representative playback synthetic can:

1. authenticate with a controlled test identity;
2. request a real manifest path;
3. validate entitlement or approved test behavior;
4. select an origin;
5. fetch a byte range from a segment;
6. validate status, headers, content length, and content bytes;
7. record each hop and total latency.

Run from multiple regions and network locations. Synthetics must be rate-limited and excluded from business analytics where appropriate.

---

## 13. Safe mitigation hierarchy

Prefer the smallest reversible action that restores the user journey.

1. **Pause the responsible rollout.**
2. **Shift traffic away from the affected region, zone, gateway, subset, node pool, or version.**
3. **Reduce retry amplification.**
4. **Apply bounded concurrency or load shedding at the caller.**
5. **Serve cached, stale, or simplified responses where product semantics allow it.**
6. **Disable nonessential personalization, recommendations, experiments, or enrichment.**
7. **Route to a healthy dependency or origin pool.**
8. **Scale the proven bottleneck**, not every caller.
9. **Correct a specific timeout mismatch** only after confirming the operation is healthy but legitimately longer than the current budget.
10. **Rollback the faulty configuration or binary.**

Avoid:

- increasing every timeout;
- enabling retries at multiple layers;
- restarting all Pods before collecting evidence;
- scaling callers into a saturated dependency;
- marking probes deep enough to create a fleet-wide readiness collapse;
- globally bypassing caches or gateways without capacity analysis;
- declaring recovery based only on green infrastructure health checks.

---

## 14. Durable design controls

### 14.1 One deadline budget

Define the user-visible deadline and allocate it explicitly across hops. Propagate the remaining budget and cancellation.

### 14.2 One retry owner per failure class

Document which layer retries connection failures, resets, and retryable status codes. Cap attempts globally.

### 14.3 Bounded queues

Every queue and pool must have:

- maximum size;
- maximum wait time;
- rejection behavior;
- priority or request-class policy;
- saturation metric;
- owner.

### 14.4 Bulkheads

Isolate:

- playback-start from background refresh;
- small metadata from large objects;
- critical entitlement from recommendation enrichment;
- tenants or request classes where necessary;
- regions and zones;
- origin pools;
- long-running work from interactive threads.

### 14.5 Graceful degradation

Define product-approved fallbacks before the incident:

- cached manifest;
- non-personalized defaults;
- reduced bitrate or simplified response;
- stale metadata within a bounded age;
- deferred analytics;
- disabled noncritical experiment calls.

### 14.6 Correct shutdown and draining

Align:

```text
readiness removal
  -> Service / EndpointSlice update
  -> load-balancer deregistration
  -> proxy connection drain
  -> application stops new work
  -> in-flight request completion or cancellation
  -> process exit
```

The grace period must fit the maximum supported request duration plus control-plane propagation margin.

### 14.7 Load tests that reproduce real failure modes

Include:

- realistic request-class mix;
- large and small responses;
- byte ranges;
- cold starts;
- connection churn;
- retries;
- packet loss and latency;
- dependency slowdown;
- one-zone loss;
- HPA scale-out;
- rolling deployment;
- cancellation and client aborts;
- partial response stalls.

Test p99 and error budget impact, not just average throughput.

---

## 15. Decision tree

```text
504 observed
  |
  +--> Who generated it?
          |
          +--> cloud LB
          |     +--> connect / TLS timeout?
          |     +--> target idle timeout?
          |     +--> return-path or response-length issue?
          |
          +--> gateway / Envoy
          |     +--> UT / upstream_response_timeout?
          |     +--> SI / stream idle timeout?
          |     +--> pool pending or overflow?
          |     +--> retries consumed budget?
          |     +--> wrong route, cluster, or endpoint?
          |
          +--> application gateway
                +--> dependency timeout?
                +--> queue or lock wait?
                +--> object read or streaming stall?

For the failing hop:
  |
  +--> connect never completed
  +--> connected, no headers
  +--> headers arrived, body stalled
  +--> dependency completed after caller deadline
  +--> request canceled but work continued
  +--> only one route / class / cohort affected
```

---

## 16. Hands-on failure-injection lab

### Lab objective

Demonstrate at least four ways a target can remain healthy while a real request returns `504`, then prove the issuer and first exhausted deadline.

### Suggested stack

- Kubernetes;
- Envoy/Istio gateway and sidecars;
- simple API or manifest service;
- mock dependency or object origin;
- Prometheus-compatible metrics;
- OpenTelemetry tracing;
- load generator;
- optional cloud load balancer.

### Experiments

1. **Dependency queue saturation**
   - Keep `/healthz` local.
   - Limit the dependency pool to a small number.
   - Drive high concurrency.
   - Observe green health, growing pool wait, and 504s.

2. **Retry amplification**
   - Add 500 ms dependency delay.
   - Enable retries at gateway and application.
   - Compare user requests with dependency attempts.
   - assign one retry owner and retest.

3. **Stream idle timeout**
   - Return headers immediately.
   - pause body transmission longer than the proxy stream idle timeout.
   - compare small response success with large response failure.

4. **Wrong route subset**
   - Route one header or request class to an under-capacity subset.
   - Keep default health route on the healthy subset.
   - identify the difference through actual proxy config.

5. **Packet loss / MTU sensitivity**
   - Inject controlled packet loss or reduce MTU in a test environment.
   - compare tiny health responses with large payloads.

6. **Drain mismatch**
   - terminate a Pod while a long request is active.
   - use a grace period shorter than request duration.
   - correct readiness, drain, and termination timing.

7. **Deadline inversion**
   - configure the dependency timeout longer than the gateway timeout.
   - show that the gateway returns 504 while the dependency continues work.
   - propagate cancellation and shorter inner deadlines.

### Exit criteria

The learner must produce:

- an end-to-end request-path diagram;
- the exact 504 issuer;
- structured access-log evidence;
- a trace showing the first exhausted deadline;
- a timeout hierarchy;
- a request-versus-attempt calculation;
- a reversible mitigation;
- a durable prevention plan;
- an external business synthetic.

---

## 17. Principal-level 90-second answer

> I would first identify which hop generated the 504 by correlating the request ID across the edge, cloud load balancer, Envoy gateway, sidecars, application, and dependency. I would capture response flags, response-code details, upstream host, attempt count, duration, bytes sent, and time to first byte. A green health check only proves that its specific path, host, port, payload, and timeout succeeded.
>
> Then I would find the first exhausted deadline: connect, TLS handshake, upstream request, per-try, stream idle, connection-pool wait, application dependency, or storage read. I would compare small and large requests, hits and misses, new and reused connections, regions, zones, routes, versions, and request classes. I would also inspect queue depth, pool pending requests, retries, retransmissions, locks, and cancellation because low CPU can hide saturation.
>
> The immediate mitigation is to pause the rollout, shift traffic away from the affected cohort, reduce retries, cap concurrency, and degrade noncritical features while protecting the dependency. I would only extend a timeout if the operation is healthy and the deadline hierarchy is wrong. The durable fix is propagated deadlines and cancellation, one retry owner, bounded queues, bulkheads, graceful drain, structured proxy logs, and a synthetic that completes the real playback path—including a segment byte-range read.

---

## 18. Likely follow-up questions

### Why does 504 not identify the root cause?

It identifies the gateway that stopped waiting. The upstream may itself be blocked on another service, queue, lock, network path, or storage operation.

### Why can health checks remain green?

They may use a shallow endpoint, different route, different protocol, smaller payload, no authentication, no dependency fan-out, and very low concurrency.

### What is the first metric you inspect?

There is no single universal metric. I first identify the issuer, then correlate its timeout count with access-log duration, response details, upstream host, queue/pool wait, and the trace span consuming the deadline.

### Why not increase the ALB or gateway idle timeout?

If the upstream is saturated, longer waits increase in-flight concurrency and resource retention. It can convert fast failures into a deeper queue collapse.

### How do you distinguish request timeout from stream idle timeout?

A request timeout limits total elapsed request time. A stream idle timeout triggers when no data moves for the configured interval, even if the total allowed duration is longer.

### Why can a large segment fail while `/healthz` succeeds?

Large responses expose body streaming, flow control, object-store latency, buffering, packet loss, MTU, retransmission, and content-length behavior that a tiny health response never tests.

### What if the application completed after the gateway returned 504?

The deadline hierarchy and cancellation propagation are wrong. Inner work should stop before the outer deadline and release resources when the caller abandons the request.

### When is increasing capacity appropriate?

After proving which queue, pool, service, database, storage system, or network resource is saturated and confirming that additional capacity can become usable without overwhelming the next dependency.

---

## 19. Review checklist

Before declaring the incident resolved, confirm:

- [ ] the exact 504 issuer is known;
- [ ] one failed request was traced across every hop;
- [ ] response flags and response-code details were captured;
- [ ] connect, request, per-try, idle, and application deadlines are documented;
- [ ] inner deadlines expire before outer deadlines;
- [ ] cancellation propagates;
- [ ] retries have one owner and a total budget;
- [ ] queue and connection-pool wait are measured;
- [ ] small and large responses were compared;
- [ ] byte-range and streaming behavior were tested;
- [ ] route, subset, version, region, zone, and node cohorts were compared;
- [ ] green health checks were compared with the real business path;
- [ ] dependency capacity was protected during mitigation;
- [ ] graceful drain and termination timing were validated;
- [ ] an external business synthetic covers the user journey;
- [ ] p99/p99.9 and error-budget impact recovered;
- [ ] rollback is proven.

---

## 20. References

- Envoy: Response Code Details — https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_conn_man/response_code_details
- Envoy: Access Logging — https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage.html
- Envoy: Circuit Breaking — https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/circuit_breaking
- Kubernetes: Liveness, Readiness, and Startup Probes — https://kubernetes.io/docs/concepts/workloads/pods/probes/
- Kubernetes: Configure Liveness, Readiness, and Startup Probes — https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/
- AWS: Troubleshoot Application Load Balancers — https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-troubleshooting.html
- AWS: Check Application Load Balancer Target Health — https://docs.aws.amazon.com/elasticloadbalancing/latest/application/check-target-health.html

---

## Core principle

> Infrastructure health is necessary but not sufficient. The authoritative health signal is whether a representative user request can complete the real dependency and streaming path within its deadline—without retry amplification, unbounded queueing, or abandoned work continuing after the caller has timed out.
