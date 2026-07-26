# Principal Platform Engineering Interview Curriculum

## Chapter 6 — Business-Aware Kubernetes Probes, Graceful Degradation, and Safe Traffic Admission

## 1. Original interview question

> A Kubernetes service passes its liveness and readiness checks, but customers still experience failures because one or more critical business capabilities are degraded. Design a probe and traffic-admission strategy that reflects real user impact without causing cascading restarts or removing too much capacity.

---

## 2. What the interviewer is testing

This question is not mainly about remembering the syntax of `livenessProbe`, `readinessProbe`, or `startupProbe`.

It tests whether you understand:

- the difference between process health, traffic eligibility, and business correctness;
- why liveness must be conservative;
- why readiness should represent whether the instance can safely serve a class of traffic;
- how dependency failures should influence admission decisions;
- how to avoid turning a shared dependency outage into a fleet-wide restart storm;
- how to support partial service and graceful degradation;
- how Kubernetes Services, EndpointSlices, kubelet probes, ingress, service mesh, and load balancers interact;
- how probe timing affects rollout safety, capacity, and incident behavior;
- how to validate probe design with production evidence rather than intuition.

A Staff or Principal engineer is expected to explain not only what probe configuration to use, but also the control-loop consequences of every failure signal.

---

## 3. Foundations from zero

### 3.1 A probe is an actuator input

A probe is not merely an observation. Its result drives a control loop.

- A failed liveness probe causes the kubelet to restart the container.
- A failed readiness probe removes the Pod from normal Service traffic.
- A startup probe delays liveness and readiness evaluation until initialization succeeds.

Therefore the first design question is:

> What action is safe when this signal becomes false?

If the action is unsafe, the probe is wrong even if the check itself is technically accurate.

### 3.2 Three distinct health questions

A production service usually has at least three separate health dimensions.

#### Process health

Can the process make forward progress?

Examples:

- event loop is not deadlocked;
- worker pool is not permanently wedged;
- essential internal threads are alive;
- memory corruption has not made the process unusable.

This is the domain of liveness.

#### Traffic eligibility

Can this replica safely accept new requests right now?

Examples:

- initialization is complete;
- required configuration is loaded;
- local caches are warm enough;
- connection pools are available;
- the replica is not draining;
- critical local resources are below hard saturation limits.

This is the domain of readiness.

#### Business capability

Can a specific customer operation succeed with acceptable semantics?

Examples:

- browse requests can work while checkout is disabled;
- playback metadata can be served while recommendations are unavailable;
- read-only operations can continue while writes are paused;
- cached responses can be served while the origin is degraded.

This often requires capability-specific routing, not one binary readiness bit.

### 3.3 Why one `/health` endpoint is usually insufficient

A single endpoint often mixes incompatible concerns:

```text
/health
  checks process
  checks database
  checks cache
  checks external API
  checks message broker
  checks disk
```

This creates dangerous coupling.

If the database is unavailable and liveness checks the database, Kubernetes may restart every healthy application process even though restart cannot repair the database. The restart storm then destroys connection reuse, increases bootstrap load, lengthens recovery, and removes diagnostic evidence.

If readiness checks every optional dependency, one noncritical failure may remove all replicas and turn partial degradation into total outage.

The core principle is:

> A probe should test conditions that the resulting Kubernetes action can improve or safely contain.

---

## 4. Assumptions

For the interview scenario, assume:

- a stateless HTTP service runs in Kubernetes;
- traffic arrives through an ingress or service mesh;
- the service has several downstream dependencies;
- some dependencies are critical for all requests;
- others are required only for specific capabilities;
- the service can provide degraded responses for some operations;
- autoscaling and rolling deployment are enabled;
- customer traffic is large enough that removing many replicas at once can cause overload.

We will distinguish four dependency classes:

1. **Local mandatory state** — configuration, credentials, listener sockets, internal workers.
2. **Global mandatory dependency** — required for nearly every request.
3. **Capability-specific dependency** — required only for one feature or endpoint.
4. **Optional enrichment dependency** — failure should not remove the replica from service.

---

## 5. Interview-ready answer

> I would separate process liveness, traffic readiness, startup completion, and business-capability health. Liveness would be intentionally shallow and would only fail when restarting the container is likely to restore forward progress, such as an unrecoverable event-loop deadlock or failed essential worker. It would not depend on remote databases, caches, DNS, or third-party APIs.
>
> Startup would cover slow initialization, migrations local to the process, configuration loading, and cache priming so liveness does not kill the container during normal boot.
>
> Readiness would answer whether this replica can safely accept its baseline traffic. It would include local initialization, drain state, critical local saturation, and only those remote dependencies that are truly mandatory for almost every request. Remote dependency checks would use cached state, hysteresis, time budgets, and fail-open or fail-closed behavior based on business risk.
>
> For capability-specific failures, I would avoid using one Pod-wide readiness bit. I would expose capability health to the gateway or service mesh and route or reject only the affected operation. For example, keep playback metadata available while disabling recommendations, or keep reads available while rejecting writes with a clear response.
>
> I would validate the design against control-loop behavior: how many endpoints disappear, whether remaining replicas can absorb load, how rolling updates behave, and whether a dependency outage can trigger synchronized removal or restart storms. I would monitor probe transitions, endpoint counts, restart rate, readiness duration, traffic shift, saturation, and customer SLIs.

---

## 6. Component explanations

### 6.1 Liveness probe

Liveness asks:

> Is this container irrecoverably unhealthy in a way that restart can repair?

Good liveness conditions:

- event loop heartbeat has stopped beyond a bounded threshold;
- internal watchdog detects deadlock;
- essential worker supervisor cannot recreate failed workers;
- process cannot accept any work due to unrecoverable local corruption;
- application-specific progress counter has stopped while work is queued.

Bad liveness dependencies:

- database connectivity;
- DNS resolution;
- external API availability;
- message broker availability;
- cache cluster availability;
- cloud metadata endpoint;
- certificate authority availability;
- service-mesh control plane reachability.

Why these are bad: restarting one Pod does not repair the shared dependency and may worsen the outage.

### 6.2 Readiness probe

Readiness asks:

> Should this replica receive new baseline traffic?

Useful readiness conditions:

- application initialization complete;
- configuration version accepted;
- required secrets loaded and not expired;
- listener and request workers available;
- replica not in drain mode;
- local queue length below hard safety threshold;
- thread or connection pool has minimum usable capacity;
- mandatory dependency state is sufficiently recent and usable.

Readiness must be designed with fleet capacity in mind. If all replicas share the same dependency, a hard synchronous readiness check can remove the entire fleet at once.

### 6.3 Startup probe

Startup probes prevent liveness and readiness from acting during normal initialization.

Use a startup probe when:

- application initialization is slow or variable;
- caches or models must load;
- JVM warm-up is substantial;
- large configuration sets must be compiled;
- sidecar coordination delays application readiness;
- recovery after crash involves journal replay.

A startup probe is preferable to an excessively large liveness `initialDelaySeconds` because it explicitly models startup completion and allows fast liveness detection after startup.

### 6.4 EndpointSlice and traffic removal

When readiness becomes false, the Pod's ready condition changes. Kubernetes controllers update Service endpoint data, typically represented through EndpointSlices. Proxies, kube-proxy implementations, ingress controllers, or service-mesh data planes then stop routing new traffic to the endpoint.

This propagation is not instantaneous. During transition:

- existing connections may remain open;
- stale endpoint information may persist briefly;
- external load balancers may still send traffic;
- clients may retry against the same endpoint;
- sidecars may have independent drain behavior.

Therefore readiness alone is not a complete graceful-shutdown mechanism.

### 6.5 Graceful termination

A safe shutdown sequence is:

1. Receive `SIGTERM`.
2. Mark the replica unready or enter explicit drain mode.
3. Stop accepting new business work.
4. Allow endpoint removal to propagate.
5. Drain in-flight requests.
6. Flush buffers and checkpoints.
7. Close downstream connections deliberately.
8. Exit before `terminationGracePeriodSeconds` expires.

A `preStop` hook may help create a propagation delay, but it must not become the only mechanism. The application should understand shutdown and drain state directly.

---

## 7. Mechanical execution

### 7.1 Baseline Kubernetes configuration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: catalog-api
spec:
  replicas: 12
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 2
  minReadySeconds: 20
  template:
    metadata:
      labels:
        app: catalog-api
    spec:
      terminationGracePeriodSeconds: 45
      containers:
        - name: app
          image: registry.example.com/catalog-api:1.42.0
          ports:
            - name: http
              containerPort: 8080
          startupProbe:
            httpGet:
              path: /health/startup
              port: http
            periodSeconds: 2
            timeoutSeconds: 1
            failureThreshold: 60
          livenessProbe:
            httpGet:
              path: /health/live
              port: http
            periodSeconds: 10
            timeoutSeconds: 1
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /health/ready
              port: http
            periodSeconds: 5
            timeoutSeconds: 1
            successThreshold: 2
            failureThreshold: 3
          lifecycle:
            preStop:
              httpGet:
                path: /admin/drain
                port: http
```

This configuration is only a starting point. Timing values must be based on measured startup, failure-detection, and endpoint-propagation behavior.

### 7.2 Suggested endpoint contracts

#### `/health/live`

Returns success when:

- process supervisor is alive;
- event loop or request dispatcher advances;
- essential internal workers are making progress.

It performs no network calls.

#### `/health/startup`

Returns success when:

- configuration is loaded;
- required secrets are present;
- listeners are bound;
- local state is initialized;
- startup-only validation is complete.

#### `/health/ready`

Returns success when:

- startup completed;
- replica is not draining;
- local resource safety thresholds are satisfied;
- minimum mandatory dependency capability is available or within an accepted stale-state window.

#### `/capabilities`

Returns a structured internal view, for example:

```json
{
  "baseline": "available",
  "search": "available",
  "recommendations": "degraded",
  "checkout": "unavailable",
  "configVersion": "2026-07-24.18",
  "observedAt": "2026-07-24T16:30:00Z"
}
```

This endpoint should normally be consumed by internal routing, diagnostics, or control systems, not exposed directly to end users.

### 7.3 Cached dependency state

Do not make every readiness probe execute a fresh synchronous request to every dependency.

Instead:

```text
background dependency monitor
        |
        +--> bounded request with jitter
        +--> circuit breaker state
        +--> last-success timestamp
        +--> recent latency/error window
        |
        +--> cached capability state
                 |
                 +--> readiness endpoint
                 +--> request routing logic
                 +--> metrics
```

Benefits:

- avoids probe-driven load amplification;
- prevents all replicas from probing at the same instant;
- supports hysteresis;
- allows stale-but-safe operation;
- separates dependency observation from kubelet timing.

### 7.4 Hysteresis

A healthy system should not flap readiness on one transient failure.

Example state machine:

```text
READY
  -- 3 consecutive critical failures --> SUSPECT
SUSPECT
  -- failure persists for 15 seconds --> NOT_READY
SUSPECT
  -- 2 successes --> READY
NOT_READY
  -- 5 consecutive successes over 20 seconds --> READY
```

The exact values depend on request volume, capacity reserve, failure modes, and recovery characteristics.

---

## 8. Scaling mathematics

### 8.1 Capacity after readiness removal

Assume:

- 100 replicas;
- each safely handles 500 requests per second;
- normal traffic is 35,000 requests per second;
- 40 replicas become unready because a shared dependency check fails.

Remaining capacity:

```text
60 replicas × 500 requests/second = 30,000 requests/second
```

Traffic exceeds safe capacity by 5,000 requests per second.

The readiness action converts a dependency degradation into overload of the remaining fleet.

A Principal engineer must therefore evaluate:

```text
remaining safe capacity >= admitted traffic + retry traffic + rollout reserve
```

### 8.2 Probe traffic amplification

Assume:

- 2,000 Pods;
- readiness probe every 5 seconds;
- each readiness request checks 4 dependencies synchronously.

Probe-triggered downstream rate:

```text
2,000 / 5 × 4 = 1,600 dependency checks per second
```

During a dependency outage, those requests may time out, consume sockets, and synchronize retries. A cached background monitor with jitter is safer.

### 8.3 Detection time

Approximate readiness failure detection time:

```text
periodSeconds × failureThreshold + request timeout effects
```

For `periodSeconds: 5`, `failureThreshold: 3`, and one-second timeouts, the endpoint may remain eligible for roughly 11–16 seconds depending on scheduling and failure timing.

This is not inherently bad. Faster removal is not always safer. Detection speed must balance customer errors against capacity collapse and transient noise.

### 8.4 Startup budget

For a startup probe:

```text
maximum startup window = periodSeconds × failureThreshold
```

Example:

```text
2 seconds × 60 = 120 seconds
```

The budget should exceed a high percentile of legitimate startup time, but not hide a permanently broken initialization indefinitely.

---

## 9. Failure-mode tree

```text
Customers fail while probes are green
|
+-- probe checks wrong layer
|   +-- listener works but business path fails
|   +-- static 200 response detached from application internals
|   +-- sidecar answers probe while application is wedged
|
+-- dependency classification wrong
|   +-- mandatory dependency omitted
|   +-- optional dependency treated as mandatory
|   +-- capability-specific dependency collapsed into global readiness
|
+-- stale or misleading state
|   +-- health cache never expires
|   +-- circuit breaker reports healthy after credential expiry
|   +-- configuration loaded but invalid for current traffic
|
+-- capacity failure
|   +-- local queue saturated
|   +-- thread pool exhausted
|   +-- connection pool exhausted
|   +-- memory pressure causes extreme latency before OOM
|
+-- network or mesh mismatch
|   +-- kubelet probe bypasses sidecar path
|   +-- service traffic requires mTLS but probe does not
|   +-- ingress route broken while Pod-local check succeeds
|   +-- DNS or discovery failure not exercised by probe
|
+-- rollout behavior
|   +-- readiness becomes true before warm-up
|   +-- minReadySeconds absent
|   +-- maxUnavailable removes too much capacity
|   +-- startup probe budget too short
|
+-- shutdown race
    +-- Pod remains ready after SIGTERM
    +-- external load balancer propagation delay
    +-- long-lived connections not drained
    +-- termination grace too short
```

---

## 10. Debugging workflow

### Step 1 — Confirm what customers experience

Start with the customer operation, not the probe.

Determine:

- affected endpoint or capability;
- error code and body;
- latency percentile;
- geographic, tenant, device, or version scope;
- whether failures affect all Pods or a subset;
- whether retries succeed.

### Step 2 — Inspect Pod health and restart history

```bash
kubectl get pods -n production -l app=catalog-api -o wide
kubectl describe pod -n production <pod>
kubectl get events -n production --sort-by=.lastTimestamp
```

Look for:

- readiness transitions;
- liveness failures;
- restart counts;
- OOM kills;
- startup probe failures;
- termination events.

### Step 3 — Inspect EndpointSlices

```bash
kubectl get endpointslice -n production \
  -l kubernetes.io/service-name=catalog-api -o yaml
```

Compare:

- desired replicas;
- running Pods;
- ready endpoints;
- terminating endpoints;
- zone distribution.

### Step 4 — Execute probes from multiple network positions

```bash
kubectl exec -n production <pod> -- curl -fsS localhost:8080/health/live
kubectl exec -n production <pod> -- curl -fsS localhost:8080/health/ready
kubectl run -n production probe-debug --rm -it \
  --image=curlimages/curl -- \
  curl -sv http://catalog-api/health/ready
```

Also test through:

- sidecar listener;
- ClusterIP;
- ingress or gateway;
- representative client path.

A Pod-local 200 does not prove the full request path works.

### Step 5 — Compare probe behavior with business SLIs

Correlate:

- probe success ratio;
- ready endpoint count;
- request success rate;
- dependency error rate;
- saturation;
- queue depth;
- circuit-breaker state;
- retry volume;
- rollout events.

### Step 6 — Inspect application implementation

Verify that probe handlers:

- do not return a hardcoded 200;
- use bounded execution time;
- do not block on overloaded worker pools;
- do not allocate heavily;
- cannot deadlock behind the same locks as business traffic;
- expose reason codes in logs and metrics;
- do not leak secrets or dependency details externally.

### Step 7 — Validate control-loop consequences

Before changing a probe, answer:

- How many Pods could become unready simultaneously?
- Can the remaining fleet absorb traffic?
- Will clients retry and multiply load?
- Will an HPA add Pods that fail the same readiness check?
- Will a rollout deadlock because new Pods never become ready?
- Will a PodDisruptionBudget block node maintenance?

---

## 11. Immediate mitigation

Depending on the incident:

- disable or relax a harmful remote dependency readiness check;
- stop a rollout;
- route only affected capabilities away;
- activate cached or degraded responses;
- shed optional work;
- reduce retry attempts and tighten retry budgets;
- increase capacity only if new replicas can actually become ready;
- temporarily fail open for low-risk reads;
- fail closed for safety, authorization, payment, or correctness-sensitive writes;
- increase termination grace if requests are being cut off;
- restore last known good probe configuration.

A probe change during an incident is itself risky. Use a bounded canary and observe endpoint count and customer SLIs before broad rollout.

---

## 12. Permanent design

### 12.1 Define a health contract

For each service, document:

| Signal | Question | Action |
|---|---|---|
| Startup | Has initialization completed? | Delay other probes |
| Liveness | Can restart repair this local failure? | Restart container |
| Readiness | Can replica safely accept baseline traffic? | Remove from Service |
| Capability | Which business operations are available? | Route/degrade selectively |
| Saturation | Is replica approaching unsafe load? | Shed, scale, or throttle |

### 12.2 Classify dependencies

For every dependency, record:

- which operations require it;
- whether cached data is acceptable;
- maximum staleness;
- fail-open or fail-closed policy;
- timeout and retry budget;
- whether outage should affect readiness;
- whether a restart can improve the condition.

### 12.3 Separate binary readiness from capability routing

Kubernetes readiness is binary per Pod. Real systems often need multiple capabilities.

Options include:

- separate Deployments and Services per capability;
- route-level health in an API gateway;
- service-mesh routing based on cluster or subset health;
- explicit feature disablement;
- read/write endpoint separation;
- degraded response modes;
- cell-based isolation.

Do not overload Pod readiness to represent every business feature.

### 12.4 Design for overload

A readiness endpoint should not simply mark a Pod unready at the first sign of load, because removing it sends more work to peers.

Use layered protection:

1. admission limits;
2. bounded queues;
3. concurrency control;
4. load shedding;
5. priority classes for critical operations;
6. autoscaling based on leading indicators;
7. readiness removal only at a hard safety boundary.

### 12.5 Roll out probe changes progressively

Probe configuration can cause outages without changing application code.

Use:

- configuration review;
- synthetic failure tests;
- canary Deployment;
- small initial replica percentage;
- automated endpoint-count guardrails;
- rollout pause on restart or readiness anomalies;
- rollback tested in advance.

---

## 13. Common wrong answers

### Wrong answer: "Readiness should check every dependency"

Why it fails: optional dependency failure can remove the whole fleet and destroy graceful degradation.

### Wrong answer: "Liveness should fail whenever the service cannot serve requests"

Why it fails: many request failures are caused by shared dependencies that restart cannot repair.

### Wrong answer: "Use the same endpoint for liveness and readiness"

Why it fails: restart safety and traffic-admission safety are different questions.

### Wrong answer: "Set failureThreshold to 1 for fast detection"

Why it fails: transient latency or packet loss can flap large portions of the fleet.

### Wrong answer: "The HPA will compensate for unready Pods"

Why it fails: new Pods may depend on the same failed service and never become ready; scaling may amplify bootstrap load.

### Wrong answer: "A 200 from localhost proves the service is healthy"

Why it fails: customer traffic may traverse DNS, mesh, TLS, gateway, authorization, and downstream systems not covered by localhost.

### Wrong answer: "Remove overloaded Pods from readiness"

Why it fails: this can transfer load to peers and cause a readiness death spiral. Shed work before removing capacity.

---

## 14. Trade-offs

### Shallow readiness

Advantages:

- avoids fleet-wide removal during dependency outages;
- preserves capacity;
- reduces flapping.

Risks:

- traffic may reach replicas that cannot complete some operations;
- requires request-level error handling and graceful degradation.

### Deep readiness

Advantages:

- prevents routing to replicas with clearly unusable mandatory dependencies.

Risks:

- creates synchronized failure domains;
- probe calls can amplify dependency load;
- can remove all capacity.

### Fail open

Appropriate when:

- operation is low risk;
- stale data is acceptable;
- dependency state is likely transient;
- availability is more important than perfect freshness.

### Fail closed

Appropriate when:

- authorization cannot be verified;
- payment correctness is uncertain;
- safety policy cannot be evaluated;
- write ownership or fencing cannot be established;
- data corruption is possible.

---

## 15. Security

Health endpoints can expose sensitive information.

Avoid returning:

- database hostnames;
- secret names or values;
- internal topology;
- certificate subjects;
- stack traces;
- dependency credentials;
- tenant data.

Recommended controls:

- expose minimal status to kubelet;
- keep detailed diagnostics on an authenticated administrative endpoint;
- bind admin endpoints to a separate port or interface;
- restrict access using network policy and service-mesh authorization;
- log reason codes internally;
- ensure probes do not bypass required application security in a way that masks real failures.

---

## 16. Observability

Track at least:

### Probe metrics

- liveness successes and failures;
- readiness successes and failures;
- startup duration;
- readiness transition count;
- time spent unready;
- probe-handler latency;
- probe reason code.

### Kubernetes metrics

- ready versus desired replicas;
- EndpointSlice ready endpoint count;
- restart rate;
- rollout availability;
- Pod termination duration;
- unavailable replicas by zone.

### Application metrics

- request success and latency by capability;
- dependency error rate;
- queue depth;
- worker utilization;
- connection-pool saturation;
- circuit-breaker state;
- degraded-response rate;
- load-shed count.

### Alerts

Good alerts correlate probe state with customer impact.

Examples:

- ready endpoints below safe capacity and traffic remains high;
- readiness transitions spike across multiple zones;
- customer error rate rises while readiness remains green;
- restart rate rises during a downstream outage;
- rollout adds Pods but ready endpoint count does not increase;
- termination duration exceeds grace budget.

---

## 17. 90-second answer

> I separate startup, liveness, readiness, and business capability because they drive different actions. Liveness is shallow and local: it fails only when the process cannot make forward progress and restart is likely to help. It never synchronously checks a database, DNS, cache, or third-party service.
>
> Startup protects slow initialization from premature liveness restarts. Readiness determines whether the replica can safely serve baseline traffic, including initialization, drain state, and hard local saturation. For remote dependencies, I classify them as globally mandatory, capability-specific, or optional. I avoid checking every dependency synchronously on every probe; instead a jittered background monitor maintains cached health with hysteresis and bounded staleness.
>
> Because Kubernetes readiness is binary, I do not remove an entire Pod for a recommendation or enrichment outage. I use route-level capability controls, degraded responses, read/write separation, or separate Services. I also model the capacity consequence before changing readiness: if 40% of endpoints disappear, can the remaining fleet survive traffic and retries?
>
> Finally, I integrate readiness with graceful drain, minReadySeconds, rolling-update limits, EndpointSlice propagation, SLOs, and automated rollout guardrails. The design goal is to protect customers without allowing probes to convert a partial dependency failure into a restart storm or total capacity outage.

---

## 18. Deep-dive answer

A strong deep-dive answer should walk through this sequence:

1. Define customer operations and capability classes.
2. Classify each dependency.
3. Define what restart can repair.
4. Implement startup, liveness, and readiness separately.
5. Add capability state outside the binary readiness bit.
6. Add background dependency monitoring with jitter and hysteresis.
7. Quantify fleet capacity after endpoint removal.
8. Integrate shutdown and connection draining.
9. Validate rollout and failure behavior.
10. Instrument customer SLIs and probe control-loop metrics.

The interviewer should hear that health is not a Boolean property of a process. It is a contract between a workload, a traffic layer, and a customer operation.

---

## 19. Whiteboard explanation

Draw five layers:

```text
Customer operation
       |
Gateway / mesh capability routing
       |
Kubernetes Service and EndpointSlices
       |
Pod readiness and drain state
       |
Process liveness and startup
```

Then draw dependencies beside the application:

```text
mandatory for all traffic
capability-specific
optional enrichment
```

Finally show actions:

```text
local deadlock ----------> restart
unsafe baseline service -> remove endpoint
single feature failure --> degrade or route selectively
shared outage -----------> preserve capacity, shed load, reduce retries
```

---

## 20. Follow-up questions

### How do you detect a deadlock without making liveness depend on the same stuck request thread?

Use an independent watchdog, progress counter, or lightweight supervisor path that does not require the blocked worker pool or contested application lock.

### Should readiness include database connectivity?

Only when the database is mandatory for nearly all baseline requests and sending traffic to the replica is predictably unsafe. Even then, use cached state, hysteresis, and capacity analysis rather than one synchronous query per probe.

### What if some endpoints need the database and others do not?

Use capability-specific routing, separate Services or Deployments, or request-level fail-fast behavior. One Pod-wide readiness bit is too coarse.

### How do you avoid synchronized probe behavior?

Use background monitors with jitter, independent schedules, cached state, randomized retry backoff, and fleet-level rollout controls.

### What is the danger of using readiness as overload protection?

Removing one overloaded Pod sends more work to the remaining Pods, potentially causing a cascading readiness collapse. Prefer concurrency limits, bounded queues, priority, and load shedding.

### What role does `minReadySeconds` play?

It prevents a Deployment from considering a new Pod available immediately after a brief readiness success. It is useful for detecting warm-up instability before rollout proceeds.

### What if the sidecar is ready but the application is not?

The Pod readiness contract must include application readiness. Ensure probe routing and sidecar rewrite behavior are understood and tested. Do not allow a proxy-only response to mask an unhealthy application.

---

## 21. Hands-on lab

### Objective

Create a service where:

- `/health/live` reports local process progress;
- `/health/startup` remains false during initialization;
- `/health/ready` reflects drain state and mandatory baseline capability;
- `/capabilities` exposes optional feature degradation;
- a downstream failure does not restart the application;
- a capability failure does not remove the Pod globally.

### Exercises

1. Deploy 10 replicas.
2. Configure startup, liveness, and readiness probes.
3. Simulate a 45-second optional dependency outage.
4. Verify ready endpoint count remains stable.
5. Simulate loss of a mandatory baseline dependency.
6. Measure endpoint removal time.
7. Verify remaining capacity and error behavior.
8. Introduce one-second dependency latency spikes.
9. Add hysteresis and compare readiness flapping.
10. Send `SIGTERM` and confirm drain behavior.
11. Perform a rolling update with `maxUnavailable: 0`.
12. Make the new version fail readiness and observe rollout pause.
13. Add a canary guardrail based on ready endpoints and customer errors.

### Evidence to collect

- Pod events;
- restart count;
- readiness transitions;
- EndpointSlice changes;
- request success and p99 latency;
- in-flight requests during termination;
- dependency request volume caused by health monitoring;
- rollout duration and availability.

---

## 22. Mapping to your experience

For an experienced DevOps, SRE, or platform engineer, use a story involving one of these patterns:

- a shared database or DNS outage caused unnecessary Pod restarts;
- a readiness check removed too many replicas;
- a rollout passed health checks but failed real business traffic;
- an overloaded service entered a readiness death spiral;
- a graceful-shutdown race caused 502 or 503 responses;
- an optional feature failure was incorrectly treated as total service failure;
- probe timing was copied between services without measuring startup behavior.

Structure the story as:

1. Customer symptom.
2. Existing probe behavior.
3. Why the control-loop action was harmful.
4. Dependency and capability classification.
5. Probe redesign.
6. Capacity and rollout safeguards.
7. Observability added.
8. Measured result.

Example positioning:

> I have learned to treat health checks as production control inputs, not simple monitoring endpoints. In one class of incidents, the application process was healthy but a shared dependency was degraded. Deep liveness checks caused restart amplification and made recovery slower. The durable fix was to separate local progress from traffic eligibility, classify dependencies by capability, use cached health with hysteresis, and preserve partial service through fail-fast or degraded responses. I also added endpoint-count and readiness-transition guardrails so future probe changes could not silently remove unsafe amounts of capacity.

---

## Memorization card

```text
Startup: has initialization completed?
Liveness: can restart repair this local failure?
Readiness: can this replica safely serve baseline traffic?
Capability: which customer operations are available?

Never put shared remote dependencies in liveness.
Do not use one binary readiness bit for every feature.
Do not remove overloaded Pods before shedding work.
Model remaining capacity before endpoint removal.
Use cached dependency state, jitter, hysteresis, and bounded staleness.
Integrate probes with rollout, drain, EndpointSlices, SLOs, and customer impact.
```
