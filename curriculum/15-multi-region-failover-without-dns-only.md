# Chapter 15 — Multi-Region Failover Without Relying Only on DNS

## Interview scenario

A streaming platform must establish a credible multi-Region recovery path in three weeks. The public hostname cannot be changed, client releases cannot be assumed, and the design must not depend only on lowering a DNS TTL and waiting for recursive resolvers, operating systems, applications, or long-lived clients to refresh records.

The primary Region serves playback-start, manifest, entitlement, metadata, and control-plane traffic. Video objects may already be distributed through a CDN, but critical APIs, state, credentials, queues, and origin-selection systems remain Region-dependent. Leadership asks for “automatic failover” and expects the secondary Region to take traffic safely during a primary outage.

You are the Staff/Principal SRE responsible for scope, architecture, delivery, game days, failover, failback, and residual-risk communication.

> This chapter is the Netflix/media-delivery adapter and migration source. Company-neutral multi-Region and disaster-recovery foundations belong under `core/reliability/disaster-recovery/` in the Staff SRE and Platform Engineering Handbook.

---

## 1. What the interviewer is testing

A strong answer must show that you can:

- separate traffic steering from application and data recovery;
- explain why DNS-only failover is not deterministic enough for every client and outage;
- choose active-passive, warm-standby, or active-active based on state and time constraints;
- define RTO, RPO, degraded mode, write ownership, and failback before selecting products;
- prevent split brain with fencing, leases, epochs, or a single-writer authority;
- validate secondary capacity, dependencies, credentials, quotas, and data freshness;
- distinguish failover of new connections from migration of established sessions;
- build deep regional health signals instead of routing based on shallow load-balancer checks;
- design a reversible operational procedure with explicit stop conditions;
- communicate what can and cannot be made production-ready in three weeks.

The weak answer is:

> “Use a low Route 53 TTL and point the record to the other Region.”

The Principal answer is:

> “I would constrain the first release to a proven active-passive path unless the data model is already active-active. I would put both regional stacks behind a stable global front door that can steer new connections independently of recursive DNS caching. But routing is the easy part. I would define RTO and RPO, establish one write authority with fencing, replicate the minimum critical state, pre-provision and load-test secondary capacity, validate every dependency, and rehearse both failover and failback. If state integrity or dependency readiness cannot be proven in three weeks, I would deliver a scoped degraded recovery mode rather than claim full multi-Region availability.”

---

## 2. Start with the business recovery contract

Before discussing Global Accelerator, a CDN, anycast, or a database product, obtain agreement on the recovery contract.

### 2.1 Recovery Time Objective (RTO)

RTO is the maximum acceptable time from a qualifying failure until the required service level is restored.

Example:

```text
Playback-start API RTO: 5 minutes
Full personalization RTO: 30 minutes
Back-office analytics RTO: 8 hours
```

RTO must include:

- failure detection;
- decision and authorization;
- write fencing;
- traffic shift;
- capacity realization;
- cache warm-up;
- dependency validation;
- user-path verification.

A traffic-management API call that completes in 30 seconds does not prove a 30-second RTO.

### 2.2 Recovery Point Objective (RPO)

RPO is the maximum acceptable amount of data loss, usually expressed as time or committed operations.

Example:

```text
Entitlement changes: RPO <= 30 seconds
Playback-session telemetry: RPO <= 5 minutes
Recommendation impressions: RPO <= 15 minutes
```

RPO is a product and data-semantics decision. It must be validated against actual replication lag, queue backlog, acknowledgement behavior, and recovery procedures.

### 2.3 Maximum tolerable outage and degraded mode

Ask:

- Which user journey must survive?
- Which features may be disabled?
- Is read-only acceptable?
- May stale metadata be served?
- Can users re-authenticate?
- Can existing streams continue while new playback starts are impaired?
- Is a reduced catalog or bitrate acceptable?
- Which markets, devices, or account tiers are protected first?

A credible three-week project usually succeeds by reducing scope, not by pretending every workload has identical criticality.

### 2.4 Success criteria

Define measurable exit criteria such as:

```text
- 99.9% of valid playback-start attempts succeed during the game day.
- p99 manifest latency remains below the degraded-mode threshold.
- no dual-writer period occurs.
- acknowledged entitlement writes lose no more than the approved RPO.
- secondary Region sustains 120% of expected failover peak for 60 minutes.
- failback completes without duplicate processing or unresolved divergence.
```

---

## 3. “No DNS-only failover” does not mean “DNS is forbidden”

DNS remains part of normal internet naming. The requirement means DNS propagation must not be the only control used to detect failure and redirect traffic.

DNS-only failover can be delayed or made inconsistent by:

- recursive resolver caching;
- operating-system and application DNS caches;
- clients that ignore or cap TTL behavior;
- connection pools that do not resolve again while connections remain open;
- JVM, runtime, SDK, or proxy cache settings;
- negative caching;
- mobile networks and captive resolvers;
- stale CDN or proxy origin resolution;
- long-lived HTTP/2, WebSocket, QUIC, or TLS sessions;
- partial outages where the endpoint still accepts TCP but cannot serve the business transaction.

DNS is useful for coarse placement and disaster recovery. It is not a precise mechanism for moving every established connection on a fixed deadline.

---

## 4. Stable global front-door patterns

The exact product is secondary to the properties required:

- a stable hostname and preferably stable edge or anycast addresses;
- health-aware routing to regional endpoints;
- weighted and reversible traffic shifts;
- TLS identity continuity;
- independent control from regional application stacks;
- observability for decision, propagation, connection, and error behavior;
- a control path that remains available when one Region is impaired.

### 4.1 Global anycast accelerator

```text
clients
   |
   v
stable anycast addresses
   |
   +----> regional load balancer A
   |
   +----> regional load balancer B
```

Benefits:

- stable entry addresses;
- rapid steering of new flows;
- regional endpoint health and weighting;
- useful for non-HTTP as well as HTTP/TCP workloads, depending on the service.

Risks and limits:

- established flows may remain on the old path until they end or are reset;
- health checks can still be too shallow;
- a healthy regional load balancer does not prove the data plane or state is safe;
- traffic steering does not replicate data or fence writes.

### 4.2 CDN or global HTTP edge

```text
client
  |
  v
CDN / edge proxy
  |
  +--> origin group: Region A
  +--> origin group: Region B
```

Benefits:

- stable public hostname;
- edge termination and request-layer routing;
- cache and origin shielding;
- possible per-path or per-request failover;
- user-journey synthetics can validate edge-to-origin behavior.

Risks:

- origin failover behavior may differ for idempotent and non-idempotent methods;
- retries at the edge can duplicate work;
- streaming and byte-range behavior must be tested;
- stale origin configuration or cache may mask failures;
- edge health must represent application correctness, not only TCP reachability.

### 4.3 Provider-neutral anycast or global proxy tier

A dedicated global proxy or anycast network can offer provider independence and custom policy, but it introduces another production platform that must be secured, scaled, and operated across failure domains.

### 4.4 Client-side regional endpoint selection

Clients may know multiple regional endpoints or receive endpoint lists from a bootstrap service. This can provide fast local recovery but is difficult when clients cannot be updated, are intermittently connected, or retain old configurations.

### 4.5 Which pattern fits the three-week constraint?

Use the existing global front door if one already exists and is operationally trusted. Introducing a new global networking platform and a new multi-Region data model simultaneously is usually too much uncontrolled change.

---

## 5. Choose the operating model deliberately

### 5.1 Backup and restore

- lowest steady-state cost;
- longest RTO;
- restore procedure and data validation dominate;
- not suitable for minute-level recovery unless restore is highly automated and rehearsed.

### 5.2 Pilot light

- core data and minimal services remain available;
- compute scales after declaration;
- RTO includes provisioning, scheduling, warm-up, and dependency readiness.

### 5.3 Warm standby

- secondary is continuously deployed at reduced capacity;
- data replication and health verification run continuously;
- capacity scales before or during traffic shift;
- practical for a three-week scoped recovery project when infrastructure is already codified.

### 5.4 Active-passive

- one Region owns writes and normal traffic;
- secondary is ready but not authoritative;
- failover transfers authority under fencing;
- simpler consistency model than active-active;
- preferred when time is short and state is not already conflict-aware.

### 5.5 Active-active

- both Regions serve traffic concurrently;
- requires explicit data partitioning, quorum, conflict resolution, idempotency, or commutative operations;
- failover may be simpler for stateless reads but much harder for writes;
- should not be created by adding a second writer to a database that was designed for one.

**Three-week default:** active-passive or warm standby, with a documented degraded mode.

---

## 6. Reference architecture

```text
                        +--------------------------+
                        | Stable global front door |
                        | anycast / CDN / proxy    |
                        +------------+-------------+
                                     |
                       weighted, health-aware routing
                    +----------------+----------------+
                    |                                 |
          +---------v----------+            +---------v----------+
          | Region A           |            | Region B           |
          | regional LB        |            | regional LB        |
          | Kubernetes/API     |            | Kubernetes/API     |
          | local cache        |            | local cache        |
          | local dependencies |            | local dependencies |
          +---------+----------+            +----------+---------+
                    |                                  |
                    +---------- control plane ----------+
                               |
                     write-authority record
                     fencing epoch / lease
                               |
            +------------------+------------------+
            |                                     |
      primary data store                    replicated state
      and write owner                       / standby store
```

The architecture must make these decisions visible:

1. Who directs new traffic?
2. Who decides that a Region is eligible?
3. Who owns writes?
4. How is the previous writer fenced?
5. Which state is replicated, and at what lag?
6. Which dependencies are regional and which are global?
7. How does the system operate when the control plane is unavailable?
8. How are failover and failback authorized and audited?

---

## 7. State classification before replication

Inventory every stateful dependency and classify it.

| State class | Examples | Recovery concern |
|---|---|---|
| Authoritative transactional data | accounts, entitlement, billing state | no split brain, strict write ownership, known RPO |
| Session state | playback sessions, tokens, device sessions | externalize, reissue, or tolerate loss |
| Derived data | recommendations, indexes, materialized views | rebuild or lag tolerance |
| Cache | metadata, manifests, authorization cache | cold-start and stampede protection |
| Queue/stream | events, jobs, playback telemetry | producer/consumer fencing, offsets, duplicates |
| Object storage | manifests, segments, images, artifacts | replication status, version consistency, range behavior |
| Configuration | feature flags, routing, catalog policy | immutable promotion and regional parity |
| Secrets and certificates | TLS, signing keys, API credentials | independent regional access and rotation |
| Build artifacts | images, packages, charts | local availability without failed-Region dependency |

For each item document:

- source of truth;
- replication direction;
- consistency model;
- acknowledgement point;
- measured lag;
- failover operation;
- failback operation;
- conflict behavior;
- data validation method;
- owner.

---

## 8. Write authority and split-brain prevention

Traffic shifting before write fencing is unsafe.

### 8.1 Why “only one Region is routed” is not fencing

The old Region may still write because of:

- established connections;
- internal callers that bypass the global front door;
- queue consumers;
- scheduled jobs;
- operators using regional endpoints;
- asynchronous workers;
- network partition rather than complete Region loss;
- stale routing or service discovery.

### 8.2 Fencing mechanisms

Possible mechanisms include:

- a monotonically increasing writer epoch;
- lease ownership with expiry and renewal;
- database-enforced leader or primary role;
- storage-level conditional writes;
- quorum-backed authority record;
- token or credential revocation for the old writer;
- disabling old-region producer/consumer roles;
- security-policy changes that deny writes from the old Region.

A write request should carry or be validated against the current authority epoch where practical.

Example state machine:

```text
PRIMARY_A epoch=41
  |
  | declare failover
  v
FENCE_A
  |
  | prove A cannot commit new writes
  v
PROMOTE_B epoch=42
  |
  | validate data and dependencies
  v
SHIFT_TRAFFIC_TO_B
```

Never reverse the sequence to “shift first and clean up later.”

### 8.3 Queue and stream fencing

Do not forget:

- producer authority;
- consumer group ownership;
- offsets/checkpoints;
- delayed and in-flight messages;
- duplicate delivery;
- ordering;
- poison messages;
- dead-letter queues;
- cross-Region replication lag.

Consumers must be idempotent or deduplicate by operation identity. Failover is not safe merely because the database is safe.

---

## 9. Replication lag and RPO evidence

Do not advertise an RPO from a configuration setting. Measure it.

Track:

- last replicated commit timestamp;
- replication sequence/LSN/offset difference;
- bytes or events pending;
- oldest unreplicated event age;
- apply lag versus transport lag;
- replication errors;
- destination storage pressure;
- recovery validation status.

Example decision rule:

```text
Failover eligibility =
  secondary healthy
  AND replication lag <= approved RPO
  AND write fencing available
  AND required dependencies healthy
  AND capacity >= minimum failover load
```

If lag exceeds the RPO, the incident commander needs an explicit business decision:

- wait for catch-up;
- accept greater data loss;
- enter read-only mode;
- recover only a subset of services;
- restore from another source.

---

## 10. Dependency readiness is often the real blocker

The secondary Region may have healthy Pods but still be unusable because of missing dependencies.

Verify:

- quotas and service limits;
- subnet and load-balancer capacity;
- container images and package repositories;
- KMS keys and grants;
- secrets, certificates, signing keys, and trust bundles;
- identity-provider and token-validation configuration;
- third-party IP allowlists;
- egress and VPC endpoints;
- DNS and private hosted-zone behavior;
- object and artifact replication;
- queue and stream resources;
- database replicas and promotion permissions;
- observability ingestion and alert routing;
- on-call access and break-glass credentials;
- feature flags and regional configuration;
- fraud, billing, DRM, entitlement, and external APIs.

A dependency inventory should state whether each dependency is:

- Region-local;
- multi-Region;
- globally shared;
- failed open;
- failed closed;
- disabled in degraded mode;
- manually switched.

---

## 11. Capacity engineering

A secondary at 10% scale is not a warm standby if it needs 20 minutes to obtain capacity during a 5-minute RTO.

### 11.1 Capacity questions

- What fraction of global traffic must the secondary absorb?
- Is the primary already sharing traffic with another healthy Region?
- What is the failover peak, not the average?
- How much load is reduced by graceful degradation?
- Are nodes, IP addresses, load-balancer targets, NAT ports, and database connections available?
- Does autoscaling have quota and instance-type diversity?
- How long do new Pods take to become useful?
- Do local caches create origin amplification while warming?

### 11.2 Capacity equation

A simple planning model is:

```text
required secondary capacity
  = protected peak traffic
  × failover headroom factor
  ÷ degraded-mode traffic reduction
```

Do not treat the factor as universal. Validate it through load testing.

### 11.3 Pre-scale before traffic shift

Typical sequence:

1. raise minimum compute capacity;
2. verify nodes and Pod IPs;
3. scale application and dependency pools;
4. warm bounded critical caches;
5. verify connection pools and database limits;
6. run synthetic transactions;
7. shift a small traffic percentage;
8. observe before increasing weight.

---

## 12. Regional health must represent the business path

### 12.1 Shallow checks are necessary but insufficient

A load balancer may declare a Region healthy because `/healthz` returns `200` while:

- replicated data is stale;
- write authority is absent;
- entitlement fails;
- a signing key is unavailable;
- queues are not writable;
- object storage cannot serve byte ranges;
- origin selection points to the failed Region;
- a critical third party blocks the secondary egress IP;
- the Region cannot sustain meaningful concurrency.

### 12.2 Health layers

Use at least:

1. **Infrastructure health** — endpoint reachable, TLS works, process responsive.
2. **Capability health** — required credentials, state, and dependencies are available.
3. **Synthetic transaction** — real or safe test account completes the user journey.
4. **Capacity health** — Region has headroom for the traffic being assigned.
5. **Authority health** — correct write epoch/role is active.

### 12.3 Failover eligibility versus normal readiness

Do not place expensive global transactions inside every Kubernetes readiness probe. Instead, a regional controller or traffic-management system should combine authoritative signals into an eligibility decision.

Example:

```yaml
region: us-west
eligibleForReadTraffic: true
eligibleForWriteTraffic: false
replicationLagSeconds: 18
writerEpoch: 41
syntheticPlaybackStart: pass
capacityHeadroomPercent: 65
criticalDependencyFailures: []
```

---

## 13. Long-lived connections and streaming behavior

Moving new connections does not automatically move existing connections.

Account for:

- HTTP/2 connection reuse;
- WebSockets;
- QUIC/HTTP/3 connection migration behavior;
- long polling;
- gRPC streams;
- persistent SDK pools;
- ongoing playback segment requests;
- upload or telemetry streams.

Possible actions:

- allow healthy existing streams to drain;
- reset connections only when required;
- reduce max connection age before a planned game day;
- return retryable signals with bounded backoff;
- ensure clients can re-authenticate in the new Region;
- preserve idempotency keys across reconnect;
- distinguish playback continuation from new playback-start traffic.

Do not mass-reset established connections during a partial outage unless the replacement Region is proven ready.

---

## 14. Session, identity, and token design

Regional failover often exposes hidden session coupling.

Validate:

- signing and verification keys exist in both Regions;
- token issuers and audiences are consistent;
- revocation and entitlement freshness meet policy;
- opaque sessions are replicated or clients can re-authenticate;
- cookies and host/domain attributes remain valid;
- device sessions tolerate Region change;
- clock synchronization prevents token validity failures;
- KMS or HSM dependencies are Region-resilient;
- emergency credentials are audited and time limited.

Where possible, make request authorization reconstructable from durable identity and entitlement state rather than a Region-local memory object.

---

## 15. Failover runbook

A failover runbook should be executable under pressure and distinguish automatic actions from human authorization.

### Phase 0 — Declare and freeze

- establish incident command;
- define affected journeys and Regions;
- freeze unrelated deployments and data changes;
- capture replication and authority state;
- stop automation that could fight the failover;
- communicate the approved degraded mode.

### Phase 1 — Qualify the secondary

Verify:

```text
[ ] secondary control and data planes reachable
[ ] required artifacts and configuration present
[ ] replication lag within accepted RPO or exception approved
[ ] capacity pre-scaled
[ ] credentials and certificates valid
[ ] third-party allowlists confirmed
[ ] deep synthetics pass
[ ] observability and paging work
[ ] rollback path available
```

### Phase 2 — Fence the old writer

- revoke or disable old write authority;
- stop old-region queue producers/consumers as required;
- invalidate old epoch or lease;
- prove commits cannot succeed from the old Region;
- record the fencing evidence.

### Phase 3 — Promote the secondary

- promote the data layer or authority record;
- assign a new writer epoch;
- enable required workers and schedulers;
- validate read-after-write and idempotency behavior;
- run protected synthetic writes.

### Phase 4 — Shift traffic progressively

Example:

```text
0% -> 1% -> 5% -> 25% -> 50% -> 100%
```

At each stage verify:

- user SLIs;
- error-budget burn;
- replication or data errors;
- queue and database saturation;
- regional capacity;
- origin/cache load;
- retry volume;
- dependency health;
- security anomalies.

### Phase 5 — Stabilize

- keep the old Region fenced;
- preserve evidence;
- monitor delayed jobs and duplicate processing;
- reconcile data gaps;
- document customer impact and accepted data loss;
- prevent automated failback.

---

## 16. Failback is a separate migration

Failback is frequently riskier than failover because state has changed in the recovery Region.

### 16.1 Failback questions

- Is the original Region repaired or merely reachable?
- How will data written in the secondary return?
- Can replication direction be reversed safely?
- Are schemas and application versions compatible?
- How are divergent writes detected?
- What happens to queues, offsets, and delayed jobs?
- Is a read-only synchronization period required?
- Will caches and indexes be rebuilt?
- Can traffic remain in the recovery Region until the next planned window?

### 16.2 Safe failback sequence

```text
repair A
  -> validate A as non-authoritative replica
  -> synchronize and reconcile data
  -> load test A
  -> deep synthetics
  -> fence B writes
  -> promote A with a new epoch
  -> shift traffic progressively
  -> monitor
```

Never automatically fail back merely because the primary health check becomes green. Flapping between Regions can produce repeated customer impact and data corruption.

---

## 17. Automatic versus manual failover

### Automatic failover is appropriate when:

- the failure signal is authoritative and resistant to false positives;
- data is already safe for promotion;
- write fencing is automatic and proven;
- dependencies and capacity are continuously validated;
- rollback is bounded;
- the failure mode is well rehearsed.

### Human authorization is appropriate when:

- promotion may exceed the RPO;
- the old writer cannot be conclusively fenced;
- the outage is partial or ambiguous;
- third-party dependencies need manual changes;
- failover may cause irreversible data or financial consequences;
- degraded-mode trade-offs require product approval.

A strong design automates evidence gathering and mechanical steps even when the final decision remains human.

---

## 18. Three-week delivery plan

### Week 1 — Scope, architecture, and foundation

1. Select the protected user journey and degraded mode.
2. Define RTO, RPO, capacity, and data-loss approval.
3. Choose active-passive or warm standby.
4. Inventory state and dependencies.
5. establish the global front-door path.
6. Deploy the secondary from immutable IaC and identical artifacts.
7. Set up replication and measured lag.
8. Design write authority and fencing.
9. Confirm quotas, certificates, secrets, KMS, endpoints, and allowlists.
10. Draft failover and failback runbooks.

### Week 2 — Correctness, capacity, and observability

1. Implement authority state and promotion workflow.
2. Build deep regional synthetics.
3. Pre-scale and load-test the secondary.
4. Validate queues, caches, object storage, indexes, and third parties.
5. Add traffic weights and one-command rollback.
6. Instrument replication lag, capacity, business SLIs, and regional eligibility.
7. Test client reconnection and long-lived connection behavior.
8. Validate degraded-mode feature flags.
9. Run a non-production failover and failback.

### Week 3 — Progressive game days and production readiness

1. Fail one dependency in the secondary.
2. Fail one AZ and verify regional resilience.
3. Shift 1% production traffic to the secondary.
4. Simulate primary read-path failure.
5. Rehearse writer fencing and protected promotion.
6. Run a controlled regional evacuation.
7. Measure detection, decision, promotion, traffic-shift, and recovery time.
8. Reconcile data and test failback.
9. close critical findings or document accepted residual risk.
10. obtain product, security, data, and operations sign-off.

### What not to promise in three weeks

Do not promise active-active writes, zero data loss, instant movement of all established clients, or automatic failback unless those properties already exist and have been proven.

---

## 19. Observability and decision dashboard

A failover dashboard should include:

### User SLIs

- playback-start success and latency;
- manifest success and latency;
- entitlement/license success;
- segment availability and rebuffer indicators;
- error-budget burn by Region and request class.

### Traffic

- connections and requests by front-door endpoint and Region;
- traffic weights and routing decision state;
- active versus new connections;
- retries, resets, and connection failures;
- client reconnect behavior.

### Capacity

- ready and schedulable application capacity;
- node and Pod IP headroom;
- load-balancer target and connection headroom;
- cache hit/miss and origin QPS;
- queue, database, and thread/connection-pool saturation.

### Data

- replication lag and errors;
- writer Region and epoch;
- rejected stale-epoch writes;
- queue offsets and backlog;
- reconciliation discrepancies;
- RPO compliance.

### Dependencies

- identity, KMS, secrets, certificates;
- object storage and CDN origin health;
- third-party API success;
- egress and private endpoint health;
- telemetry ingestion and alert delivery.

---

## 20. Security and governance

Multi-Region recovery must not create a permanent security bypass.

Require:

- least-privilege promotion and traffic-shift roles;
- multi-party approval for destructive or data-loss decisions;
- immutable audit logs;
- short-lived emergency credentials;
- independent regional access paths;
- replicated but controlled secrets;
- tested certificate and trust rotation;
- network policies and egress controls in both Regions;
- old-writer credential revocation during fencing;
- explicit authority for accepting RPO violations;
- regular access and runbook review.

Failover permissions should be exercised during game days so they do not fail only during a real disaster.

---

## 21. Common failure modes

### 21.1 Global front door shifts traffic to an unready Region

Cause: health check validates a shallow endpoint.

Control: combine deep synthetics, authority, replication, dependency, and capacity signals.

### 21.2 Secondary cannot scale

Cause: quota, instance scarcity, subnet IP exhaustion, image-pull dependency, or autoscaler delay.

Control: pre-provision minimum failover capacity and test realistic peak load.

### 21.3 Both Regions accept writes

Cause: routing was treated as fencing.

Control: storage- or authority-enforced writer epoch and explicit old-writer revocation.

### 21.4 Replication is connected but stale

Cause: apply lag, backlog, destination saturation, or schema incompatibility.

Control: monitor data age and validation, not only replication process status.

### 21.5 Secondary depends on the failed Region

Cause: secrets, identity, artifacts, telemetry, DNS, queue, or database control plane remains primary-only.

Control: dependency graph and regional evacuation tests.

### 21.6 Failover works but failback corrupts data

Cause: reverse replication and divergent state were never designed.

Control: treat failback as a planned migration with reconciliation and new fencing epoch.

### 21.7 Retry storm follows traffic shift

Cause: cold caches, pool churn, and retries at multiple layers.

Control: staged weights, retry budgets, stale serving, load shedding, and origin protection.

### 21.8 Established clients remain on the failed path

Cause: connection reuse or long-lived sessions.

Control: drain/reset policy, client reconnect behavior, and separate new-flow versus existing-flow SLIs.

### 21.9 Automatic failover flaps

Cause: ambiguous health or insufficient hysteresis.

Control: consecutive-failure thresholds, multi-signal eligibility, hold-down periods, and manual failback.

---

## 22. Anti-patterns to name in the interview

- “Multi-Region” means only two copies of stateless Pods.
- A low DNS TTL is assumed to move all clients immediately.
- Traffic shifts before write fencing.
- Replication configured means RPO proven.
- The secondary is scaled from zero during a minute-level RTO.
- Readiness is equated with regional eligibility.
- All features are kept enabled during recovery.
- Active-active is selected without a conflict model.
- Failback is automatic when the old Region becomes reachable.
- Game days stop after traffic steering and do not test data.
- Third-party allowlists and credentials are checked during the outage.
- The control plane for failover exists only in the failed Region.
- Operators cannot distinguish read eligibility from write eligibility.

---

## 23. Hands-on game-day lab

### Lab objective

Build and prove a scoped active-passive recovery path that does not rely only on DNS propagation.

### Suggested environment

- two Kubernetes clusters or equivalent regional stacks;
- one stable global proxy or traffic-steering layer;
- primary/replica transactional store;
- replicated object store;
- queue or stream;
- regional caches;
- synthetic playback-start transaction;
- Prometheus-compatible metrics and tracing.

### Experiment 1 — Traffic steering only

- fail the primary application endpoint;
- shift new traffic to the secondary;
- observe that routing works;
- verify which stateful calls still fail;
- document why traffic movement is not full recovery.

### Experiment 2 — Stale replication

- introduce replication lag beyond the RPO;
- trigger failover eligibility evaluation;
- verify promotion is blocked or requires explicit exception;
- test read-only degraded mode.

### Experiment 3 — Split-brain prevention

- partition the primary rather than shutting it down;
- attempt to promote the secondary;
- prove old-region writes are rejected after the epoch changes;
- validate audit evidence.

### Experiment 4 — Cold secondary

- reduce secondary compute and clear regional caches;
- send 1%, 10%, and 50% traffic;
- measure warm-up, origin amplification, autoscaling delay, and p99;
- establish safe promotion gates.

### Experiment 5 — Missing dependency

- remove a secondary certificate, KMS grant, or third-party allowlist;
- ensure deep synthetic health marks the Region ineligible;
- restore the dependency and verify controlled eligibility.

### Experiment 6 — Long-lived connections

- maintain HTTP/2, gRPC, or WebSocket sessions to the primary;
- shift new flows;
- measure old-flow behavior;
- test drain and reset policy.

### Experiment 7 — Failback

- write data in the recovery Region;
- repair and resynchronize the original Region;
- validate reconciliation;
- fence the recovery writer;
- promote the original Region with a new epoch;
- shift traffic progressively.

### Required outputs

- architecture and dependency diagrams;
- RTO/RPO contract;
- state and authority matrix;
- failover and failback runbooks;
- measured timeline;
- data-loss/reconciliation report;
- stop conditions;
- residual-risk register.

---

## 24. Principal-level 90-second answer

> “I would not start with DNS or a product name. I would define the one user journey we must protect, its RTO, RPO, degraded mode, and data-loss authority. With only three weeks, I would choose warm standby or active-passive unless writes are already designed for active-active conflict handling.
>
> “I would keep the existing hostname behind a stable global front door—an anycast accelerator, CDN, or global proxy—that can steer new connections between regional load balancers without waiting only for recursive DNS caches. Then I would inventory every stateful dependency, replicate only the critical state, continuously measure lag, pre-scale and load-test the secondary, and build deep regional synthetics.
>
> “Before shifting write traffic, I would fence the primary using a writer lease or monotonic epoch that the data layer enforces. I would then promote the secondary, validate protected read/write transactions, and move traffic progressively with stop conditions on playback SLIs, replication, queues, retries, and capacity. I would keep failback manual until data is reconciled and authority can be transferred through a new epoch.
>
> “The deliverable is not a traffic switch. It is a rehearsed recovery and failback path that proves state integrity, dependency readiness, capacity, and customer impact. If those cannot be proven in three weeks, I would explicitly deliver a smaller read-only or degraded recovery mode rather than claim full production-grade multi-Region failover.”

---

## 25. Likely follow-up questions

### Why is DNS insufficient by itself?

Because resolvers, runtimes, proxies, and established connection pools may retain old addresses beyond the desired recovery time. DNS also does not validate state safety or move existing connections.

### Does a global accelerator eliminate DNS?

No. Clients still use a hostname in most architectures. The accelerator provides a stable global entry and independent steering for new flows so recovery is not dependent only on DNS record propagation.

### What is the hardest part of regional failover?

State authority, dependency readiness, capacity, and failback—not the routing API.

### How do you prevent split brain?

Use a writer lease, epoch, primary-role mechanism, or conditional authority that the storage/write path enforces. Revoke the old Region before promoting the new writer.

### Would you choose active-active?

Only when the data model already supports partitioning, quorum, idempotency, or conflict resolution. A three-week deadline is not enough to invent a safe active-active write model for a single-writer system.

### How do you handle replication lag above the RPO?

Block automatic promotion and request an explicit decision: wait, accept loss, enter read-only mode, or recover only a subset of journeys.

### How do you know the secondary is healthy?

Combine infrastructure health, deep user transactions, data freshness, write authority, dependency readiness, and capacity headroom.

### Why is automatic failback dangerous?

The recovery Region may contain newer authoritative data. Returning automatically can reverse replication incorrectly, lose writes, or cause flapping.

### What do you test with streaming workloads?

Playback start, manifests, entitlement/DRM, segment byte ranges, cache and origin behavior, long-lived connections, retries, rebuffer indicators, and client reconnection.

### What is the honest three-week deliverable?

A scoped, measured, active-passive recovery path with stable global steering, proven fencing, replicated critical state, tested capacity, and a rehearsed failback—or a clearly documented degraded mode if full recovery cannot be proven.

---

## 26. Review checklist

Before declaring the capability production-ready, confirm:

- [ ] protected user journeys and degraded modes are approved;
- [ ] RTO and RPO include detection, decision, promotion, and validation;
- [ ] a stable global front door can steer new traffic without DNS-only propagation;
- [ ] established-connection behavior is understood;
- [ ] state and dependency inventories are complete;
- [ ] replication lag is measured against the RPO;
- [ ] old-writer fencing is enforced by the write path;
- [ ] queue producers, consumers, offsets, and duplicates are addressed;
- [ ] secondary capacity is pre-provisioned or proven within RTO;
- [ ] deep regional synthetics pass;
- [ ] certificates, secrets, keys, artifacts, and allowlists work independently;
- [ ] caches and origins survive cold failover;
- [ ] traffic shifts progressively with explicit stop conditions;
- [ ] failback and reconciliation are rehearsed;
- [ ] automatic failback is disabled unless explicitly proven;
- [ ] emergency access is least-privilege and audited;
- [ ] game-day findings have owners and deadlines;
- [ ] residual risks are documented and accepted.

---

## 27. References

- AWS Global Accelerator documentation — https://docs.aws.amazon.com/global-accelerator/
- Amazon CloudFront origin failover — https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/high_availability_origin_failover.html
- AWS Disaster Recovery of Workloads on AWS — https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/
- Amazon Route 53 DNS failover — https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-failover.html
- Kubernetes multi-cluster services — https://multicluster.sigs.k8s.io/
- Google SRE Book: Managing Critical State — https://sre.google/sre-book/managing-critical-state/

---

## Core principle

> Multi-Region recovery is not a DNS record change. It is an authority transfer over traffic, state, capacity, and dependencies. The system is ready only when it can fence the old writer, prove the new Region’s data and business path, shift traffic progressively, and reconcile state before a controlled failback.
