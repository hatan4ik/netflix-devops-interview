# Chapter 14 — Establishing SLO and Error-Budget Ownership

## Interview scenario

A large streaming platform has extensive dashboards, alerts, incident reviews, and reliability work, but teams still disagree about what “reliable enough” means. Product leaders push releases because infrastructure graphs are green. Service teams claim dependencies caused their incidents. Platform/SRE teams own most alerts but cannot force roadmap decisions. Some services advertise `99.99%` availability without a defensible user journey, measurement point, or error-budget policy.

You are the Staff/Principal SRE asked to create a culture where Service Level Objectives (SLOs) are measurable, owned, and used to make release and investment decisions.

> This chapter is the Netflix/media-delivery adapter and migration source. The shared curriculum maps company-neutral SLO foundations to `core/reliability/slo/` in the Staff SRE and Platform Engineering Handbook.

---

## 1. What the interviewer is testing

A strong answer must show that you can:

- define Service Level Indicators (SLIs), SLOs, SLAs, and error budgets precisely;
- choose user-centered signals rather than convenient component metrics;
- connect reliability measurement to product risk and release behavior;
- calculate and alert on error-budget burn;
- handle dependencies, partial degradation, low traffic, and multi-region services;
- create ownership without turning SLOs into punishment;
- design governance that scales across hundreds or thousands of services;
- influence product, engineering, platform, and leadership decisions.

The weak answer is:

> “Create a dashboard for uptime and page when it drops below 99.9%.”

The Principal answer is:

> I start with a small number of critical user journeys, define good and valid events at the user-observable boundary, assign product and engineering owners, and publish an error-budget policy before the first alert fires. The SLO becomes real only when its burn changes a release, capacity, or investment decision. SRE provides the measurement and governance system; the service team and product owner remain accountable for the reliability outcome.

---

## 2. The reliability contract

### 2.1 Service Level Indicator (SLI)

An SLI is a quantitative measurement of service behavior.

For an event-based SLI:

```text
SLI = good valid events / total valid events
```

Examples:

- successful playback starts divided by valid playback-start attempts;
- manifests returned correctly and within 300 ms divided by valid manifest requests;
- video-segment requests returning the requested bytes before their deadline divided by valid segment requests;
- license requests completed before the player deadline divided by valid license requests;
- stream-minutes without unexpected rebuffering divided by eligible stream-minutes.

For a time-based SLI:

```text
SLI = good time / eligible time
```

Time-based availability can be appropriate for continuously evaluated systems, but request/event SLIs usually align more directly with user impact for high-volume APIs.

### 2.2 Service Level Objective (SLO)

An SLO is the target range for an SLI over a defined compliance window.

Example:

```text
99.95% of valid playback-start attempts complete successfully
within 4 seconds over a rolling 28-day window.
```

The statement is incomplete unless it defines:

- the user journey;
- good-event criteria;
- valid-event denominator;
- measurement source and location;
- target;
- window;
- exclusions;
- segmentation;
- owner;
- review cadence;
- error-budget policy.

### 2.3 Service Level Agreement (SLA)

An SLA is an external or contractual commitment with consequences such as credits, penalties, or termination rights. An internal SLO should normally be stricter than the SLA so the organization has an operating margin before contractual breach.

### 2.4 Error budget

The error budget is the allowed amount of not-good service within the SLO window.

```text
error-budget fraction = 1 − SLO target
```

For a request-based `99.9%` SLO:

```text
allowed bad events = total valid events × 0.001
```

For a time-based `99.9%` objective over 30 days:

```text
30 days × 24 hours × 60 minutes × 0.001
= 43.2 minutes
```

This time conversion is useful for intuition, but do not convert an event-based SLO into downtime minutes and imply false precision.

---

## 3. Begin with user journeys, not services

A user does not care that `manifest-api`, `identity-v2`, and `license-gateway` were each “99.9% available.” The user cares whether playback started and continued.

### 3.1 Candidate tier-1 streaming journeys

1. **Sign in or resume an authenticated session**
2. **Open title details**
3. **Start playback**
4. **Continue playback without avoidable interruption**
5. **Seek or change bitrate successfully**
6. **Acquire or renew a DRM license**
7. **Download content for offline playback**

Start with three to five journeys that represent the largest business and customer risk. Do not begin with hundreds of service-level SLOs.

### 3.2 Journey map

```text
player
  |
  +--> edge / CDN
  +--> authentication
  +--> entitlement
  +--> metadata
  +--> manifest generation
  +--> origin selection
  +--> DRM / license
  +--> segment delivery
  +--> telemetry confirmation
```

The journey SLI should be measured as close to the user as practical. Component SLIs are then diagnostic and ownership signals, not substitutes for journey success.

### 3.3 Product semantics matter

A technically successful response may still be bad:

- HTTP `200` with an empty or malformed manifest;
- playback starts but with the wrong audio track;
- license response arrives after the player deadline;
- a segment returns bytes that fail integrity validation;
- a fallback manifest works but omits a required entitlement;
- a stream starts only after excessive retries;
- video continues at a bitrate below the agreed experience floor.

Good-event definitions must include correctness and timeliness when users experience both.

---

## 4. Design the SLI carefully

### 4.1 Availability SLI

```text
successful valid requests / total valid requests
```

Define success explicitly. Depending on the API, these may be bad events:

- `5xx` responses;
- gateway timeouts;
- client-visible connection resets;
- syntactically valid but semantically unusable responses;
- responses completing after the user deadline.

Do not automatically count every `4xx` as good or bad. A valid authentication rejection may be correct; a `429` caused by undercapacity may be user-impacting.

### 4.2 Latency SLI

A threshold SLI is usually easier to budget than a percentile-only objective:

```text
requests completed within 300 ms / total valid requests
```

This directly counts bad events. A p99 graph is useful, but it does not itself reveal how many requests violated a business deadline across all segments.

Use multiple thresholds when the product has distinct expectations:

- `99%` within 300 ms;
- `99.9%` within 1 second.

### 4.3 Correctness SLI

Examples:

- manifests pass schema and business-rule validation;
- requested title maps to the correct asset/version;
- entitlement decision matches source-of-truth policy;
- object checksum or segment integrity is valid;
- state transition is idempotent and externally observable.

Correctness SLIs often require synthetics, audit comparison, sampled validation, or downstream confirmation rather than proxy status codes.

### 4.4 Freshness SLI

```text
responses with data age <= freshness threshold / valid responses
```

Useful for:

- catalog metadata;
- recommendation inputs;
- entitlement updates;
- configuration propagation;
- subtitle or audio availability;
- regional content-rights changes.

### 4.5 Streaming quality SLIs

Possible user-centered signals include:

- playback-start success;
- time to first frame;
- rebuffer-free stream minutes;
- unexpected playback termination;
- segment delivery before playback deadline;
- bitrate or resolution floor;
- seek success;
- license renewal success.

Do not combine every experience dimension into one opaque score. Keep independently actionable SLOs for availability, latency, correctness, and quality where needed.

---

## 5. Denominator engineering

The denominator is where many SLOs are manipulated accidentally or deliberately.

### 5.1 Valid-event rules

A valid event should represent a real opportunity for the system to serve the user.

Potential exclusions may include:

- explicitly supported test traffic;
- malformed requests that could never succeed;
- confirmed client cancellation before server processing;
- unsupported protocol versions;
- approved maintenance only when the product contract explicitly allows it.

Potentially dangerous exclusions:

- all dependency failures;
- all regional incidents;
- all overload responses;
- errors during releases;
- requests from a difficult device population;
- “known issues” with no expiration;
- telemetry gaps treated as success.

A dependency failure is usually still part of the user-facing service’s SLO. The team may use dependency SLIs for attribution, but the user journey did not succeed.

### 5.2 Unknown must not silently become good

If telemetry is missing, delayed, sampled incorrectly, or contradictory, classify it explicitly:

```text
GOOD | BAD | UNKNOWN | EXCLUDED
```

Track unknown rate and alert when it exceeds a small allowed threshold. Otherwise, an observability outage can make the SLO look better.

### 5.3 Prevent denominator dilution

High-volume easy traffic can hide a low-volume critical path.

Example:

- cached catalog reads: 10 billion requests at `99.999%`;
- playback license renewals: 1 million requests at `98%`.

A single fleet-wide ratio may look excellent while a critical journey is broken. Use separate SLOs or protected segmentation for distinct user intents.

---

## 6. Measurement points and sources of truth

### 6.1 Server-side measurement

Advantages:

- high coverage;
- consistent instrumentation;
- low dependence on client telemetry delivery;
- easy correlation with traces and deployments.

Blind spots:

- network and CDN failures before the request arrives;
- responses written by the server but never received;
- player decode or playback failures;
- client-side deadlines shorter than server completion.

### 6.2 Edge/CDN measurement

Advantages:

- closer to the user;
- captures gateway, routing, and upstream failures;
- broad geographic visibility.

Blind spots:

- semantic correctness;
- player behavior after response delivery;
- client cancellation semantics.

### 6.3 Client/player telemetry

Advantages:

- strongest view of actual playback experience;
- observes time to first frame, rebuffering, decode, and device-specific failures.

Blind spots:

- offline or blocked telemetry;
- event delivery delay;
- sampling and SDK-version bias;
- users who abandon before telemetry flushes.

### 6.4 Synthetic transactions

Advantages:

- controlled, repeatable journeys;
- useful in every region and during low traffic;
- detects semantic failure.

Blind spots:

- small sample size;
- may not represent real accounts, titles, devices, networks, or data distributions.

### 6.5 Recommended model

Use a primary source of truth plus corroborating signals:

```text
primary journey SLI: edge or client event stream
corroboration: server metrics + traces + synthetics
component diagnosis: service, proxy, dependency, and infrastructure metrics
```

Document ingestion lag, late-event policy, deduplication, sampling, and replay behavior.

---

## 7. Error-budget mathematics

### 7.1 Budget remaining

```text
allowed bad events = valid events × (1 − objective)

budget remaining = allowed bad events − observed bad events
```

Example:

```text
objective:         99.95%
valid events:      2,000,000,000
allowed bad rate:  0.0005
allowed bad events: 1,000,000
observed bad events: 250,000
budget remaining:   750,000 (75%)
```

### 7.2 Burn rate

Burn rate compares the observed bad-event rate with the allowed bad-event rate.

```text
burn rate = observed bad-event rate / allowed bad-event rate
```

For a `99.9%` objective, the allowed bad-event rate is `0.1%`.

If the observed bad-event rate is `1%`:

```text
burn rate = 1% / 0.1% = 10×
```

At a steady `10×` burn rate, a 30-day budget is consumed in roughly three days.

### 7.3 Time to exhaustion

An approximation:

```text
time to exhaustion ≈ compliance-window duration / burn rate
```

Assuming the current rate continues and no budget has already been consumed.

### 7.4 Budget is not “allowed outage time” to spend casually

The budget expresses uncertainty and accepted risk. It is not permission to schedule avoidable incidents or consume the final percentage before doing reliability work.

---

## 8. Multi-window burn-rate alerting

A single alert on SLO attainment is too late. By the time a 28- or 30-day objective crosses its threshold, the budget may already be exhausted.

Use paired windows:

- a short window confirms current impact;
- a longer window prevents noise from brief spikes;
- page only when both conditions are true.

Illustrative policy for a 30-day window:

| Severity | Long window | Short window | Burn threshold | Approximate budget consumed |
|---|---:|---:|---:|---:|
| Page | 1 hour | 5 minutes | 14.4× | 2% in 1 hour |
| Page | 6 hours | 30 minutes | 6× | 5% in 6 hours |
| Ticket | 1 day | 2 hours | 3× | 10% in 1 day |
| Ticket | 3 days | 6 hours | 1× | 10% in 3 days |

These thresholds are starting points, not universal constants. Adjust for traffic volume, detection delay, recovery time, service tier, and business risk.

### 8.1 Prometheus-style recording rules

Illustrative request availability rules:

```yaml
groups:
  - name: playback-start-slo
    interval: 30s
    rules:
      - record: journey:playback_start_valid:rate5m
        expr: |
          sum(rate(playback_start_total{classification!="excluded"}[5m]))

      - record: journey:playback_start_bad:rate5m
        expr: |
          sum(rate(playback_start_total{classification="bad"}[5m]))

      - record: journey:playback_start_error_ratio:rate5m
        expr: |
          journey:playback_start_bad:rate5m
          /
          clamp_min(journey:playback_start_valid:rate5m, 1)

      - record: journey:playback_start_burn_rate:rate5m
        expr: |
          journey:playback_start_error_ratio:rate5m / 0.001
```

A paired alert:

```yaml
- alert: PlaybackStartFastErrorBudgetBurn
  expr: |
    (
      journey:playback_start_error_ratio:rate1h / 0.001 > 14.4
    )
    and
    (
      journey:playback_start_error_ratio:rate5m / 0.001 > 14.4
    )
  for: 2m
  labels:
    severity: page
    service_tier: tier1
  annotations:
    summary: Playback-start error budget is burning rapidly
    runbook: https://runbooks.example.net/playback-start-slo
```

Production rules must handle absent metrics, low traffic, label consistency, recording-rule evaluation, and aggregation carefully.

### 8.2 Low-traffic services

Ratios become unstable with tiny denominators. Combine ratio and event-count conditions, use synthetics, or evaluate over longer windows.

Example condition:

```text
page only if burn threshold is exceeded
AND bad-event count exceeds a minimum
AND valid-event count is sufficient
```

Do not hide one catastrophic failure in a critical low-volume administrative path merely because the count is small; choose the alert policy based on impact.

---

## 9. Aggregation and segmentation

### 9.1 Fleet-wide SLO plus protected slices

A global SLO may be useful for executive risk, but it can hide regional or cohort failures.

Protect important slices such as:

- region and country;
- device family or player SDK version;
- subscription tier;
- new versus existing sessions;
- title or asset class without exposing high-cardinality identifiers;
- codec or DRM system;
- application version;
- critical accessibility journeys.

Do not create a separate formal SLO for every label combination. Use a primary SLO plus diagnostic or minimum-floor policies for important slices.

### 9.2 Unequal traffic across regions

Traffic-weighted global aggregation can make a small region disappear. Use both:

- global event-weighted SLO;
- regional minimum reliability floor;
- regional burn alerts;
- customer-impact counts.

### 9.3 Do not average percentages

This is wrong:

```text
(region A SLI + region B SLI) / 2
```

Unless the business intentionally weights regions equally. For event-weighted aggregation:

```text
sum(good events across regions) / sum(valid events across regions)
```

The weighting decision must be explicit.

---

## 10. Dependencies and shared platforms

### 10.1 The user-facing service owns the journey

A service cannot exclude a failed dependency and still claim the user journey succeeded. However, attribution is necessary for prioritization and accountability.

Maintain:

- journey SLO and error budget;
- service contribution to bad events;
- dependency SLIs/SLOs;
- request-volume and criticality assumptions;
- dependency contracts and escalation paths.

### 10.2 Error-budget allocation is not automatic multiplication

If a journey requires five serial dependencies, assigning every dependency the same `99.9%` objective does not produce a `99.9%` journey. Under simplistic independence assumptions:

```text
0.999^5 ≈ 99.50%
```

Real failures are often correlated, so naive probability multiplication is not a complete design method. Use architecture, fallback, redundancy, and measured contribution analysis.

### 10.3 Platform SLOs

Shared platform teams should publish service contracts for capabilities such as:

- Kubernetes scheduling and Pod readiness;
- service discovery and DNS;
- mesh configuration convergence;
- identity and certificate issuance;
- artifact delivery;
- telemetry ingestion;
- secrets delivery;
- regional traffic steering.

A platform SLO should describe the capability consumed by internal customers, not only the health of platform components.

Example:

```text
99.99% of valid certificate requests for healthy workload identities
produce a usable certificate within 30 seconds over 28 days.
```

---

## 11. The error-budget policy

An SLO without a policy is a graph. The policy defines what changes as budget health changes.

### 11.1 Example policy states

| State | Budget / burn condition | Expected behavior |
|---|---|---|
| Healthy | More than 50% budget remaining and no sustained high burn | Normal releases; continue planned reliability work |
| Watch | 25–50% remaining or recurring medium burn | Review top consumers; tighten canaries; prioritize preventive work |
| At risk | Less than 25% remaining or high burn | Require reliability review for risky releases; reduce change scope |
| Exhausted | Budget depleted | Pause discretionary high-risk changes; execute recovery plan; leadership exception required |
| Recovering | Burn controlled after exhaustion | Validate fixes and rebuild margin before returning to normal risk |

The policy should not block:

- security fixes;
- incident mitigation;
- rollback;
- changes that demonstrably reduce reliability risk.

### 11.2 Exceptions

A release exception should include:

- named business sponsor;
- engineering and SRE risk assessment;
- expected budget impact;
- canary scope;
- stop conditions;
- rollback owner and tested mechanism;
- expiration time;
- post-release review.

Exceptions must be visible and auditable, not informal pressure applied in a chat room.

### 11.3 Error budgets are not punishment

Do not use budget exhaustion to shame teams or reduce performance ratings. That drives metric manipulation and hidden incidents. Use it as a neutral risk-management mechanism.

---

## 12. Ownership model

### Product owner

- defines critical user journeys and acceptable impact;
- participates in SLO target selection;
- chooses degraded-mode semantics;
- accepts or rejects release risk;
- funds reliability work when evidence requires it.

### Service engineering team

- owns implementation and operational outcomes;
- maintains instrumentation and runbooks;
- responds to burn alerts;
- performs root-cause and recurring-consumer analysis;
- includes reliability work in planning.

### Platform/SRE

- provides SLO tooling, templates, recording rules, dashboards, and coaching;
- validates measurement quality;
- operates governance and review forums;
- helps design burn alerts and policies;
- challenges unsafe exceptions;
- does not become the permanent owner of every application SLO.

### Data/observability team

- owns ingestion integrity, lateness, deduplication, retention, and query reliability;
- publishes telemetry quality indicators;
- prevents silent denominator or schema changes.

### Leadership

- resolves conflicts between delivery pressure and reliability evidence;
- enforces error-budget policy consistently;
- funds cross-team remediation;
- evaluates repeatability and learning, not zero-incident theater.

### Security, legal, and support

- security defines non-negotiable controls and emergency-change requirements;
- legal/commercial teams align SLAs with measurable internal SLOs;
- support contributes customer-impact signals and validates whether technical metrics match complaints.

---

## 13. SLO specification as code

Store SLO definitions with version control, review, ownership, and deployment automation.

Illustrative specification:

```yaml
apiVersion: reliability.example.io/v1
kind: ServiceLevelObjective
metadata:
  name: playback-start-success
  owner: playback-platform
  productOwner: playback-experience
  tier: tier1
spec:
  description: Valid playback attempts that reach first frame within 4s
  objective: 0.999
  window: 28d
  indicator:
    type: eventRatio
    source: player-event-stream
    good: |
      event="playback_start"
      and outcome="first_frame"
      and duration_ms <= 4000
    valid: |
      event="playback_start"
      and classification != "excluded"
    unknown: |
      telemetry_state="incomplete"
  segments:
    protected:
      - region
      - device_family
      - player_major_version
  policy:
    healthyBudgetPercent: 50
    atRiskBudgetPercent: 25
    exhaustedAction: restrict-high-risk-releases
  links:
    dashboard: https://observability.example.net/slo/playback-start
    runbook: https://runbooks.example.net/playback-start
    serviceCatalog: https://catalog.example.net/playback-platform
```

Validation should reject:

- missing owners;
- absent runbook;
- no unknown-data policy;
- invalid or unbounded labels;
- impossible targets without review;
- undefined exclusions;
- missing measurement source;
- duplicate or conflicting SLO IDs.

---

## 14. Governance without bureaucracy

### 14.1 SLO review questions

1. What user journey does this protect?
2. Where is success observed?
3. What exactly is good, bad, excluded, and unknown?
4. Is the target based on user need and engineering economics?
5. Which failure classes are hidden by aggregation?
6. Who owns burn response and roadmap trade-offs?
7. What decision changes when budget health changes?
8. Can the telemetry itself fail or drift?
9. Does the service have a tested degradation mode?
10. How will the SLO evolve as the product changes?

### 14.2 Service launch gate

A tier-1 launch should require:

- approved SLO specification;
- instrumented valid/good events;
- burn-rate alerts tested with synthetic data;
- runbook and escalation ownership;
- capacity model;
- rollback and degraded mode;
- telemetry-quality monitoring;
- initial target review date.

### 14.3 Quarterly review

Review:

- budget consumed by cause;
- repeated incident classes;
- false positives and detection gaps;
- exclusions and unknown rate;
- target appropriateness;
- action-item completion;
- dependency contribution;
- release behavior changed by the policy;
- customer complaints compared with SLI results.

The goal is not to renegotiate the target downward whenever it is missed. The goal is to discover whether the product contract, measurement, architecture, capacity, or operating model needs change.

---

## 15. Adoption plan

### First 30 days — establish credibility

- select three to five tier-1 journeys;
- map each journey end to end;
- inventory existing telemetry and data gaps;
- define draft good/valid/unknown events;
- assign engineering and product owners;
- back-test candidate SLOs against recent incidents;
- publish an initial error-budget policy;
- avoid executive scorecards until data quality is credible.

### Days 31–60 — make it operational

- deploy recording rules and dashboards;
- add multi-window burn alerts;
- connect alerts to runbooks and incident tooling;
- add SLO review to release and planning forums;
- implement SLO specifications as code;
- establish exception workflow;
- train service teams using historical incidents.

### Days 61–90 — scale and govern

- onboard additional tier-1 services;
- publish platform capability SLOs;
- automate service-catalog ownership and compliance checks;
- report top budget consumers and repeated causes;
- measure telemetry unknown rate;
- integrate SLO evidence into quarterly investment planning;
- create an executive view focused on user journeys, not component counts.

### Beyond 90 days

- expand to tier-2 services using reusable templates;
- refine targets from measured user and business outcomes;
- automate release gates carefully;
- test SLO pipelines and alert policies through game days;
- deprecate vanity availability metrics;
- align external SLAs with proven internal margins.

---

## 16. Common failure modes and anti-patterns

### 16.1 Setting every target to 99.99%

Targets must reflect user need, architecture, cost, and dependency reality. More nines can require disproportionately more engineering and operational complexity.

### 16.2 Using CPU, Pod readiness, or load-balancer health as the SLO

These are diagnostic signals. They do not directly measure a successful user journey.

### 16.3 One SLO per microservice before journey SLOs

This produces hundreds of dashboards with no shared product outcome.

### 16.4 SRE owns all reliability

The service team then owns feature velocity while SRE owns consequences. This does not scale.

### 16.5 Budget exhaustion causes an absolute release freeze

Rigid freezes can block security fixes and reliability improvements. Use a risk policy with explicit exceptions and change classification.

### 16.6 Excluding dependencies

The metric becomes green while users fail. Attribute dependency contribution; do not erase user impact.

### 16.7 Treating telemetry absence as success

An observability outage improves the SLO. Track unknown separately.

### 16.8 Alerting directly on monthly SLO attainment

The alert fires after the budget is already gone. Alert on burn rate.

### 16.9 High-cardinality segmentation

Raw title IDs, customer IDs, request IDs, or cache keys can make the metric system unusable. Use bounded dimensions and logs/traces for detailed investigation.

### 16.10 Percentile averaging

Averaging p99 values from regions or time buckets is not a valid global p99. Aggregate the histogram distribution or compute the event-threshold SLI from raw/recorded counts.

### 16.11 SLO target becomes a performance score

Teams hide incidents, manipulate exclusions, and choose easy indicators. Evaluate decision quality, learning, and systemic improvement instead.

### 16.12 No action follows the dashboard

If budget health never changes roadmap, rollout, or investment decisions, the SLO program is decorative.

---

## 17. Handling difficult cases

### 17.1 New service with no baseline

Start with a provisional target based on product need and architectural evidence. Review after a defined observation period. Do not wait indefinitely for perfect history.

### 17.2 Third-party dependency

Measure the user journey, the dependency call, fallback success, and contribution to bad events. Contractual claims do not replace observed behavior. Design timeout, retry, fallback, and provider-diversity strategy around the journey SLO.

### 17.3 Planned maintenance

Whether maintenance is excluded is a product and contract decision. If users cannot use the product, excluding every planned event can make the SLO meaningless. Prefer architectures and procedures that preserve the journey through maintenance.

### 17.4 Partial degradation

A request can be:

- fully good;
- good with approved fallback;
- degraded but usable;
- bad;
- unknown.

Decide whether approved fallback counts as good based on product semantics. Also track fallback rate so the organization does not permanently normalize degraded behavior.

### 17.5 Multiple customer tiers

Different contracts or experience requirements may justify separate SLOs, but avoid hiding underperformance by moving difficult traffic into a weaker tier without a real product distinction.

### 17.6 Global incident with telemetry loss

Use independent edge, client, synthetic, support, and infrastructure evidence. Mark unknown explicitly and avoid claiming compliance until data is reconciled.

---

## 18. Incident integration

During an incident, display:

- affected journey SLO;
- current and projected burn rate;
- budget consumed;
- region/device/version slices;
- top error classification;
- deployment and configuration markers;
- dependency contribution;
- customer-impact events or minutes;
- telemetry unknown rate.

Incident command uses the SLO to prioritize:

- which journey to preserve;
- which feature to degrade;
- when to roll back;
- whether retries are helping;
- whether traffic should shift;
- when executive/customer communication is required.

Postmortems should state:

- SLI impact;
- error-budget consumption;
- detection delay;
- whether the alert policy worked;
- whether an exclusion or telemetry gap distorted impact;
- recurring failure class;
- systemic actions, owner, due date, and verification method.

---

## 19. Measuring whether the culture changed

Useful program-level indicators:

- percentage of tier-1 journeys with approved SLO, owner, policy, and runbook;
- percentage of SLOs with measured unknown-data rate;
- percentage of incidents mapped to a journey SLI;
- time from harmful burn to detection;
- proportion of pages generated by user-impact burn rather than component symptoms;
- repeat-incident rate;
- action-item completion and verified effectiveness;
- risky releases modified or stopped because of budget evidence;
- reliability work funded before budget exhaustion;
- customer complaints that are not explained by current SLIs;
- number and age of policy exceptions;
- false-positive and false-negative alert review.

Do not make “number of SLOs created” the primary success metric.

---

## 20. Hands-on lab

### Lab objective

Build and operate a playback-start SLO that changes engineering behavior.

### Data model

Generate events with:

```text
timestamp
region
device_family
player_version
outcome
first_frame_duration_ms
error_class
classification (good|bad|unknown|excluded)
release_version
```

### Exercises

1. **Define the indicator**
   - Write good and valid-event expressions.
   - Document every exclusion.
   - Add an unknown-data metric.

2. **Back-test the target**
   - Replay 60 days of synthetic history.
   - Compare `99.9%`, `99.95%`, and `99.99%` objectives.
   - Identify which historical incidents would have exhausted the budget.

3. **Build burn alerts**
   - Implement fast and slow paired windows.
   - Inject a 20-minute severe failure.
   - Inject a three-day low-level regression.
   - Verify page versus ticket behavior.

4. **Test aggregation failure**
   - Keep global reliability high while failing a low-volume region.
   - Add regional floors and verify detection.

5. **Test telemetry loss**
   - Drop 15% of client events.
   - Confirm unknown rises rather than SLO improving.

6. **Exercise policy**
   - Simulate 20% budget remaining before a major release.
   - Produce a release-risk decision with canary, stop conditions, and exception workflow.

7. **Dependency attribution**
   - Make DRM responsible for 60% of bad playback starts.
   - Preserve journey impact while producing a dependency contribution report.

### Exit criteria

The learner must deliver:

- a complete SLO specification;
- formulas and recording rules;
- burn-rate alerts;
- dashboard with protected slices;
- error-budget policy;
- ownership/RACI decision;
- release exception example;
- postmortem budget-impact section;
- 90-day adoption plan.

---

## 21. Principal-level 90-second answer

> I would not start with hundreds of service dashboards. I would select three to five tier-1 user journeys—such as sign-in, playback start, and uninterrupted streaming—and define good, valid, excluded, and unknown events at the closest reliable user-observable boundary. Each SLO gets a product owner, engineering owner, measurement source, target, window, protected slices, runbook, and error-budget policy.
>
> I would calculate burn rate rather than wait for the monthly target to fail, using paired short and long windows so severe outages page quickly and slow regressions create planned work. Dependencies remain part of the journey impact; we attribute their contribution but do not remove their failures from the denominator. Missing telemetry becomes unknown, not success.
>
> The cultural mechanism is the error-budget policy. Healthy budget permits normal delivery. Sustained burn tightens canaries and prioritizes reliability. Exhaustion restricts discretionary high-risk changes while allowing security fixes, rollback, and reliability improvements. SRE supplies tooling and governance, but product and service teams own the trade-off. I know the program works when budget evidence changes a release or investment decision and repeat incidents fall.

---

## 22. Likely follow-up questions

### Why not choose a 100% SLO?

It leaves no error budget, makes every imperfection a breach, and usually drives either unsustainable cost or dishonest exclusions. Safety-critical subfunctions may require extremely high objectives and defense in depth, but “100%” still does not create a failure-free distributed system.

### Who chooses the target?

Product, engineering, and SRE together. Product defines acceptable user impact; engineering explains architecture and cost; SRE provides risk and operability evidence.

### Should dependency failures count against our SLO?

They count against the user journey if the user failed. Track dependency attribution separately and use contracts, fallback, or architecture to control the contribution.

### What happens when the budget is exhausted?

Apply the pre-agreed policy: restrict discretionary high-risk releases, execute a reliability recovery plan, and require visible leadership approval for exceptions. Continue security, rollback, and reliability-improving changes.

### Why use burn-rate alerts instead of CPU alerts?

CPU is a cause or saturation signal. Burn rate measures the rate at which the user reliability contract is being consumed. Diagnostic alerts may still exist, but paging should be dominated by actionable user impact.

### How do you stop teams gaming exclusions?

Version-control definitions, require product/SRE review, publish excluded and unknown rates, audit changes, back-test against incidents, and forbid retroactive exclusions without explicit governance.

### Can one SLO cover availability and latency?

A threshold event can define “successful and within deadline,” but separate SLIs are often more actionable. Avoid one opaque score that hides whether failures are errors, slowness, correctness, or freshness.

### How do you handle a small region?

Use a global event-weighted SLO plus regional floors or burn alerts. Do not let high-volume regions mask a complete low-volume regional outage.

### What is the biggest organizational mistake?

Making SRE responsible for reliability while product teams retain unilateral release authority. Ownership and decision rights must align.

---

## 23. Review checklist

Before declaring an SLO production-ready, confirm:

- [ ] it protects a named user journey;
- [ ] good and valid events are unambiguous;
- [ ] exclusions are minimal, reviewed, and measurable;
- [ ] unknown telemetry is tracked separately;
- [ ] the measurement point is documented;
- [ ] target and window are explicit;
- [ ] protected slices prevent masking;
- [ ] error-budget math is validated;
- [ ] multi-window burn alerts are tested;
- [ ] low-traffic behavior is defined;
- [ ] product and engineering owners are named;
- [ ] runbook and incident integration exist;
- [ ] error-budget policy changes decisions;
- [ ] exceptions are visible and expiring;
- [ ] dependency attribution does not erase journey impact;
- [ ] SLO-as-code validation is automated;
- [ ] quarterly review and target revision rules exist;
- [ ] customer/support evidence is compared with the SLI;
- [ ] the telemetry pipeline has its own integrity indicators.

---

## 24. References

- Google SRE Book — Service Level Objectives: https://sre.google/sre-book/service-level-objectives/
- Google SRE Workbook — Implementing SLOs: https://sre.google/workbook/implementing-slos/
- Google SRE Workbook — Alerting on SLOs: https://sre.google/workbook/alerting-on-slos/
- OpenSLO specification: https://openslo.com/
- Prometheus alerting rules: https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/
- Prometheus recording rules: https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/

---

## Core principle

> An SLO is not a dashboard target owned by SRE. It is a versioned product reliability contract with a measurable user boundary, named decision owners, and an error-budget policy that governs how the organization trades delivery risk for customer trust.
