# Chapter 10 — HPA Not Scaling Under High Reported CPU

## Interview scenario

A high-traffic service is breaching latency and error-rate objectives. Dashboards show high CPU, but the Kubernetes `HorizontalPodAutoscaler` does not add replicas—or it computes more replicas but the workload never gains usable capacity. The HPA object exists, application pods are running, and the incident coincides with a major playback or API traffic spike.

You are the Staff/Principal SRE responsible for diagnosis, containment, recovery, and redesign.

---

## 1. What the interviewer is testing

A strong answer must show that you can:

- distinguish monitoring CPU from the CPU metric actually consumed by the HPA;
- explain the autoscaling control loop and its dependency chain;
- separate replica calculation, scale-subresource mutation, pod creation, scheduling, readiness, and traffic admission;
- reason about resource requests, sidecars, startup behavior, and missing metrics;
- recognize when CPU is the wrong demand signal;
- avoid destabilizing feedback loops during an outage;
- design autoscaling around user-visible service objectives rather than a single infrastructure graph.

The weak answer is: “Restart Metrics Server” or “lower the CPU target.”

The Principal answer is:

> First determine whether the HPA failed to observe demand, failed to calculate a larger replica count, failed to write the scale target, or successfully scaled replicas that never became schedulable, ready, or useful. Preserve that distinction throughout the incident.

---

## 2. Build the end-to-end autoscaling mental model

A CPU-based HPA is not one component. It is a chain:

```text
application CPU consumption
        ↓
kubelet resource metrics
        ↓
Metrics Server
        ↓
metrics.k8s.io aggregated API
        ↓
HPA controller reconciliation
        ↓
scale subresource on Deployment/StatefulSet
        ↓
workload controller creates Pods
        ↓
scheduler finds capacity
        ↓
container images start
        ↓
startup/readiness gates pass
        ↓
Service/EndpointSlice admits traffic
        ↓
load balancer actually distributes demand
```

A failure at any stage can appear to an operator as “HPA is not scaling.”

The first diagnostic question is therefore:

> Which transition in the chain did not happen?

---

## 3. Understand the core HPA calculation

For a metric target, the simplified replica calculation is:

```text
desiredReplicas = ceil(
  currentReplicas × currentMetricValue / desiredMetricValue
)
```

For CPU `averageUtilization`, Kubernetes compares usage with **CPU requests**, not CPU limits and not node CPU capacity.

Example:

```text
current replicas: 10
average CPU usage: 900m per Pod
average CPU request: 1000m per Pod
current utilization: 90%
target utilization: 60%

desired replicas = ceil(10 × 90 / 60) = 15
```

Now change only the request:

```text
average CPU usage: 900m
average CPU request: 4000m
current utilization: 22.5%
target utilization: 60%
```

The dashboard can honestly show “900m CPU per Pod,” while the HPA correctly decides not to scale because the utilization relative to the request is low.

### 3.1 Requests are part of the control system

Resource requests are not merely scheduler hints. For utilization-based HPA metrics, they are the denominator of the control equation.

Bad requests produce bad autoscaling behavior:

- requests set too high suppress scale-up;
- requests set too low cause early or excessive scale-up;
- inconsistent requests across pod versions make rollout behavior unpredictable;
- missing requests can make utilization undefined;
- default requests injected by a `LimitRange` may silently alter behavior.

### 3.2 Tolerance prevents small oscillations

The HPA avoids scaling when the metric ratio is sufficiently close to 1.0. A graph slightly above target does not guarantee a replica change.

### 3.3 The displayed status is not the complete internal decision

The HPA handles missing metrics and not-yet-ready pods conservatively. The value shown in HPA status may not reflect every damping adjustment used for the final scaling decision.

---

## 4. Define “high CPU” precisely

During the incident, never accept “CPU is high” as a complete statement.

Ask:

- Which metric source produced the graph?
- Is it container CPU, pod CPU, process CPU, node CPU, throttled time, or CPU saturation?
- Is it instantaneous, rate-based, averaged, or percentile-based?
- Is the graph in cores, millicores, or percent?
- Percent of what: one core, the CPU request, the CPU limit, or node capacity?
- Does it include the service-mesh, logging, security, or telemetry sidecars?
- Is the graph aggregated by pod, deployment, container, node, or cluster?
- What is the scrape interval and query window?
- Is the HPA reading the same time series?

A Prometheus dashboard and Metrics Server can both be correct while showing materially different values because they have different pipelines, collection intervals, aggregation, labels, and intended use.

---

## 5. The four primary failure domains

Classify the incident before changing configuration.

### Domain A — Observation failure

The HPA cannot obtain a valid metric.

Examples:

- Metrics Server is unavailable;
- the `metrics.k8s.io` APIService is unhealthy;
- kubelet metrics cannot be scraped;
- certificates or network paths between Metrics Server, kubelets, and API server are broken;
- target pods have missing CPU metrics;
- custom/external metrics adapters are returning errors or stale data.

Expected signs:

- HPA target shows `<unknown>`;
- `ScalingActive=False`;
- events contain `FailedGetResourceMetric`, `FailedGetPodsMetric`, or `FailedGetExternalMetric`;
- `kubectl top` fails or lacks affected pods.

### Domain B — Calculation or policy failure

Metrics are available, but the controller does not recommend more replicas.

Examples:

- actual utilization is below target because requests are high;
- the HPA is targeting the wrong workload;
- labels or metric selectors select the wrong population;
- `maxReplicas` has already been reached;
- a scale-up behavior policy limits growth;
- multiple metrics produce confusing status;
- not-yet-ready and missing metrics dampen scale-up;
- the metric spike is shorter than the collection and reconciliation windows;
- the controller is within tolerance.

### Domain C — Scale mutation or controller failure

The HPA recommends more replicas but cannot update the target.

Examples:

- the scale target no longer exists;
- the HPA references the wrong API version or kind;
- admission policy rejects scale changes;
- control-plane authorization or API availability fails;
- another controller or GitOps process continuously writes a conflicting replica count;
- a deployment tool reapplies a fixed `spec.replicas` value;
- two autoscalers fight over the same target.

Expected signs:

- `AbleToScale=False`;
- HPA events show failed scale retrieval or update;
- desired replicas change in HPA status but workload replicas are repeatedly reverted.

### Domain D — Capacity realization failure

The HPA scales the desired replica count, but useful serving capacity does not increase.

Examples:

- new pods remain `Pending` due to insufficient CPU or memory;
- node affinity, topology spread, taints, or quotas prevent scheduling;
- Cluster Autoscaler cannot add nodes or is slower than demand growth;
- images pull slowly;
- startup takes several minutes;
- readiness probes fail;
- sidecars fail to initialize;
- EndpointSlices do not include the new pods;
- load balancing is sticky or highly uneven;
- downstream dependencies cap useful concurrency;
- new replicas consume CPU warming caches without serving traffic.

In this domain the HPA may be functioning correctly. The service still fails because scaling is not equivalent to capacity.

---

## 6. First-response incident strategy

### Step 1 — Protect users before tuning the controller

Apply the smallest safe containment measures appropriate to the service:

- shed optional work;
- reject low-priority traffic early;
- reduce expensive feature paths;
- enable bounded queueing;
- lower per-pod concurrency to prevent collapse;
- serve stale or cached responses where correctness allows;
- reserve capacity for control, authentication, and playback-critical traffic;
- manually raise replicas when there is verified cluster headroom.

Do not wait for a perfect autoscaling diagnosis while the service exhausts itself.

### Step 2 — Establish the four replica numbers

Capture:

```bash
kubectl get hpa <name> -n <namespace> -o yaml
kubectl get deploy <name> -n <namespace> -o yaml
```

Record:

1. HPA `currentReplicas`;
2. HPA `desiredReplicas`;
3. workload `spec.replicas`;
4. workload `status.availableReplicas`.

These numbers immediately classify the failure:

| Observation | Likely domain |
|---|---|
| desired equals current despite load | observation or calculation |
| desired is greater than current but `spec.replicas` does not change | scale mutation/control plane |
| `spec.replicas` increases but pods remain Pending | scheduler/cluster capacity |
| pods run but are not Ready | startup/readiness |
| pods are Ready but latency remains high | traffic distribution, dependency, or wrong scaling signal |

### Step 3 — Read HPA conditions and events

```bash
kubectl describe hpa <name> -n <namespace>
kubectl get events -n <namespace> \
  --field-selector involvedObject.kind=HorizontalPodAutoscaler \
  --sort-by=.lastTimestamp
```

Focus on:

- `AbleToScale`;
- `ScalingActive`;
- `ScalingLimited`;
- current metrics;
- desired replicas;
- `lastScaleTime`;
- event reasons and message text.

### Step 4 — Compare the exact metric pipeline

```bash
kubectl top pods -n <namespace> --containers
kubectl get --raw '/apis/metrics.k8s.io/v1beta1/namespaces/<namespace>/pods' | jq
kubectl get apiservice v1beta1.metrics.k8s.io -o yaml
kubectl describe apiservice v1beta1.metrics.k8s.io
```

Compare these values with:

- pod CPU requests;
- the HPA target type;
- Prometheus or APM graphs;
- the timestamp of the traffic spike.

### Step 5 — Inspect requests for every relevant container

```bash
kubectl get deploy <name> -n <namespace> \
  -o jsonpath='{range .spec.template.spec.containers[*]}{.name}{" request="}{.resources.requests.cpu}{" limit="}{.resources.limits.cpu}{"\n"}{end}'
```

Check:

- app container;
- Envoy or other mesh proxy;
- logging agent;
- security agent;
- metrics sidecar;
- init containers when startup is abnormal;
- mutations from admission controllers.

### Step 6 — Determine whether new replicas can become useful

```bash
kubectl get pods -n <namespace> -l app=<label> -o wide
kubectl get pods -n <namespace> -l app=<label> \
  -o custom-columns='NAME:.metadata.name,PHASE:.status.phase,READY:.status.containerStatuses[*].ready,NODE:.spec.nodeName'
kubectl get endpointslice -n <namespace> -l kubernetes.io/service-name=<service> -o yaml
kubectl get events -n <namespace> --sort-by=.lastTimestamp
```

Inspect:

- Pending reasons;
- readiness transitions;
- image pulls;
- mount failures;
- sidecar readiness;
- topology imbalance;
- endpoint admission;
- per-pod request rate.

### Step 7 — Freeze conflicting automation

Pause or identify:

- GitOps reconciliation that pins replicas;
- scheduled scaling jobs;
- custom autoscalers;
- deployment pipelines that reapply manifests;
- VPA modes that mutate requests;
- rollout controllers that continuously replace pods.

Do not let competing writers hide the root cause.

---

## 7. Common root causes in depth

### 7.1 CPU requests make utilization look low

Symptoms:

- dashboards show high absolute CPU;
- `kubectl top` confirms usage;
- HPA current CPU percentage remains below target;
- no HPA errors appear.

Root cause:

```text
utilization = usage / request
```

A 1-core usage value against a 4-core request is only 25% utilization.

Recovery:

- do not immediately slash requests during an outage;
- manually scale if capacity exists;
- verify historical usage, startup demand, and SLO behavior;
- adjust requests through load-tested right-sizing;
- consider `AverageValue` when absolute CPU is the intended signal.

### 7.2 A container lacks a CPU request

For pod-level resource utilization, missing relevant requests can prevent a valid pod utilization calculation.

Common sources:

- newly added sidecar;
- injected proxy after a mesh upgrade;
- debugging container accidentally made permanent;
- inconsistent Helm values;
- admission mutation applied only in some namespaces.

Durable correction:

- enforce requests through policy;
- validate rendered manifests in CI;
- use `ContainerResource` metrics to scale on the primary application container when sidecar demand should not control replicas.

### 7.3 HPA reads a different CPU story than the dashboard

Examples:

- dashboard sums application plus sidecar CPU, while HPA uses a single container metric;
- dashboard shows node CPU saturation;
- dashboard query includes terminated pods;
- a five-minute Prometheus rate smooths a spike that Metrics Server samples differently;
- the graph displays percent of CPU limit rather than request;
- labels join multiple environments.

Diagnosis requires comparing exact raw values and timestamps, not screenshots.

### 7.4 HPA reached `maxReplicas`

The HPA may be calculating a larger desired value but constrained by `maxReplicas`.

Signs:

- `ScalingLimited=True`;
- current replicas equal `maxReplicas`;
- errors and latency continue increasing.

Do not blindly raise the maximum. First verify:

- node capacity;
- downstream limits;
- database connection pools;
- queue partitions;
- license or rate limits;
- cost guardrails;
- whether each new pod adds real throughput.

### 7.5 Scale-up policy is too restrictive

An `autoscaling/v2` HPA can constrain scale-up using `behavior.scaleUp.policies`.

A policy designed to prevent cost spikes can be too slow for bursty streaming traffic. Compare:

- demand growth rate;
- pod startup time;
- allowed replica growth per period;
- available headroom before saturation.

The system must begin scaling early enough that ready capacity arrives before the current fleet crosses its safe concurrency limit.

### 7.6 New pods are not yet ready, damping the calculation

The HPA treats missing metrics and not-yet-ready CPU samples conservatively. Long startup, readiness flapping, or heavy warm-up can dampen scale-up.

Typical pattern:

1. load rises;
2. HPA adds pods;
3. new pods consume high CPU warming caches;
4. readiness remains false or oscillates;
5. metrics are set aside or conservatively handled;
6. old pods remain overloaded;
7. scale-up appears slow.

Corrective design:

- use a `startupProbe` for initialization;
- make readiness represent ability to serve useful traffic;
- avoid readiness checks that depend on every remote dependency;
- reduce synchronized warm-up;
- pre-warm safely;
- keep enough baseline replicas for the startup duration.

### 7.7 Metrics Server is healthy globally but broken for one cohort

Possible causes:

- Metrics Server cannot reach kubelets on a new node pool;
- kubelet serving certificates fail validation;
- security groups or NetworkPolicy block the path;
- node addresses are selected incorrectly;
- affected nodes have clock or kubelet problems;
- aggregated API registration is healthy while scrapes partially fail.

Compare metrics availability by node, zone, node pool, and pod cohort.

### 7.8 HPA and GitOps fight over replicas

A common anti-pattern:

```text
HPA writes replicas = 30
GitOps reapplies replicas = 10
HPA writes replicas = 30
GitOps reapplies replicas = 10
```

Symptoms:

- repeated scale events;
- deployment replica count oscillates;
- GitOps drift alerts;
- no stable capacity increase.

Durable correction:

- remove ownership of `spec.replicas` from the deployment manifest when HPA owns scale;
- configure ignore-differences rules carefully;
- define a single writer for each control field;
- alert on conflicting mutations.

### 7.9 HPA scales, but the scheduler cannot place pods

Common scheduling blockers:

- insufficient allocatable CPU or memory;
- requests too large for available node shapes;
- strict anti-affinity;
- topology spread constraints;
- taints without tolerations;
- node selectors with no matching nodes;
- namespace quota;
- exhausted IP addresses;
- storage topology or volume attachment limits.

The HPA is not a node-capacity system. It can request pods that the cluster cannot run.

### 7.10 Cluster Autoscaler is slower than the traffic spike

End-to-end response time includes:

```text
HPA detection
+ HPA reconciliation
+ unschedulable pod detection
+ node provisioning
+ node bootstrap
+ image pull
+ application startup
+ readiness
+ traffic ramp
```

For a fast traffic spike, this can be much longer than the service's overload survival time.

Principal-level mitigations:

- maintain node and pod headroom;
- use overprovisioning pods;
- keep critical images cached;
- reduce bootstrap and startup time;
- forecast scheduled events;
- scale on leading indicators such as queue depth or concurrency;
- use regional admission control before saturation.

### 7.11 Sidecar CPU distorts pod-level scaling

A service-mesh or telemetry sidecar can consume significant CPU due to encryption, retries, logging, or high connection churn.

Pod-level CPU scaling can be appropriate when sidecar CPU grows proportionally with useful traffic. It can be misleading when:

- sidecar retry storms create CPU without useful work;
- telemetry export is failing;
- a proxy regression burns CPU independently of demand;
- app CPU is low but sidecar CPU is high.

Use `ContainerResource` metrics when the primary app container is the intentional control signal, but retain separate alerts for sidecar saturation.

### 7.12 CPU is not the bottleneck

Latency can rise while CPU remains misleadingly low or high.

Potential bottlenecks:

- request queues;
- event-loop saturation;
- lock contention;
- memory pressure and GC;
- connection-pool exhaustion;
- database throughput;
- downstream rate limiting;
- network bandwidth;
- disk I/O;
- single-threaded work inside a multi-core request allocation.

Scaling on CPU is useful only when CPU has a stable relationship with user demand and additional replicas increase useful throughput.

---

## 8. Multi-metric HPA semantics

With multiple metrics, Kubernetes calculates a desired replica count for each and normally chooses the largest recommendation.

This is useful for combining signals such as:

- CPU utilization;
- request concurrency;
- queue depth;
- requests per second;
- lag per consumer;
- an external demand forecast.

Operational nuance:

- a failing metric does not necessarily block scale-up if another valid metric recommends more replicas;
- an unavailable metric can block a scale-down that other metrics recommend;
- each metric must have a clear ownership and freshness SLO;
- correlated noisy metrics can amplify oscillation;
- business metrics require correct cardinality and workload-to-metric association.

Do not add metrics merely to appear sophisticated. Each signal must answer a distinct capacity question.

---

## 9. A production-grade HPA example

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: playback-api
  namespace: streaming
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: playback-api
  minReplicas: 12
  maxReplicas: 120
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      selectPolicy: Max
      policies:
        - type: Percent
          value: 100
          periodSeconds: 60
        - type: Pods
          value: 12
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      selectPolicy: Min
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
        - type: Pods
          value: 5
          periodSeconds: 60
  metrics:
    - type: ContainerResource
      containerResource:
        name: cpu
        container: playback-api
        target:
          type: Utilization
          averageUtilization: 60
```

This is an example, not a universal template.

The values must be derived from:

- per-pod safe throughput;
- startup duration;
- demand burst shape;
- cluster headroom;
- downstream capacity;
- error-budget policy;
- cost constraints;
- load-test evidence.

### Why use `ContainerResource` here?

It isolates the application container from mesh and telemetry sidecars. That is correct only when application CPU is the intended demand signal.

### Why retain a conservative scale-down?

Fast scale-down can cause:

- cache loss;
- connection churn;
- synchronized load transfer;
- repeated scale oscillation;
- increased tail latency;
- renewed node scaling.

Scale-up and scale-down should be intentionally asymmetric.

---

## 10. Capacity mathematics the candidate should articulate

### 10.1 Required replicas from throughput

```text
required replicas = peak demand / safe per-pod throughput
```

Use **safe** throughput, not benchmark maximum.

Example:

```text
peak demand: 48,000 requests/second
safe pod throughput: 800 requests/second
required serving pods: 60
```

Then add failure and rollout headroom.

### 10.2 Startup coverage

If pods need 120 seconds to become useful and demand grows by 400 requests/second each second:

```text
additional demand during startup = 120 × 400 = 48,000 requests/second
```

The current fleet needs enough headroom, queue capacity, or load shedding to survive that interval.

### 10.3 Little's Law for concurrency

For a stable system:

```text
concurrency ≈ throughput × latency
```

If latency rises, in-flight concurrency rises even at constant throughput. A concurrency metric can therefore provide a better overload signal than CPU for some services.

### 10.4 Scale-up must outrun demand growth

Define:

```text
capacity arrival rate > demand growth rate
```

Capacity arrival rate includes both replica creation rate and time-to-readiness. A fast HPA policy is useless when pods take ten minutes to start.

---

## 11. Observability requirements

Build one timeline containing:

### Demand

- incoming requests or events per second;
- concurrency;
- queue depth or consumer lag;
- request mix and cost distribution.

### User impact

- success rate;
- latency percentiles;
- timeouts;
- playback-start failures or rebuffering indicators;
- rejected and shed requests.

### HPA control state

- current replicas;
- desired replicas;
- maximum replicas;
- current metric values;
- metric targets;
- HPA conditions;
- scale events;
- time since last scale.

### Pod realization

- requested, scheduled, running, ready, and available replicas;
- startup duration;
- image-pull duration;
- readiness transition time;
- EndpointSlice membership.

### Cluster capacity

- allocatable versus requested CPU and memory;
- Pending pods;
- node provisioning time;
- subnet/IP capacity;
- quota and topology constraints.

### Efficiency

- per-pod throughput;
- per-pod CPU;
- CPU throttling;
- sidecar/app CPU split;
- downstream calls per request;
- useful work versus retries.

A single CPU graph is not an autoscaling dashboard.

---

## 12. Alerting and SLOs for the autoscaling system

Useful alerts include:

- HPA at `maxReplicas` while service SLOs degrade;
- `ScalingActive=False` for a production workload;
- desired replicas greater than available replicas for too long;
- metric age beyond an accepted freshness limit;
- high percentage of target pods missing metrics;
- Pending replicas after HPA scale-up;
- p95 startup-to-ready time exceeding the design assumption;
- repeated replica oscillation;
- GitOps or another controller mutating HPA-owned replicas;
- scale-up latency exceeding the workload's overload survival window.

Define an autoscaling SLO such as:

> During a valid demand increase, the control system must produce sufficient ready capacity before the service consumes its permitted error budget.

That SLO must be decomposed into observation, decision, scheduling, startup, readiness, and traffic-admission latency.

---

## 13. Safe mitigation hierarchy

Prefer reversible actions with bounded blast radius.

1. **Protect the service**
   - load shed;
   - reduce optional work;
   - protect critical traffic classes.

2. **Add known-safe capacity**
   - manually scale within verified cluster and dependency headroom;
   - increase node capacity when scheduling is the blocker;
   - temporarily reserve warm capacity.

3. **Repair the metric path**
   - restore Metrics Server or adapter availability;
   - repair kubelet connectivity;
   - correct selectors and missing requests.

4. **Repair the control policy**
   - adjust a proven-too-low maximum;
   - correct scale-up policies;
   - remove conflicting replica ownership.

5. **Redesign the signal**
   - move from CPU to concurrency, queue, lag, or a composite strategy when load tests show CPU is not predictive.

Avoid during the incident unless evidence demands it:

- mass pod restarts;
- deleting the HPA without recording state;
- setting extremely low CPU targets;
- unbounded `maxReplicas` increases;
- removing resource requests;
- disabling readiness to force endpoints into service;
- scaling beyond downstream capacity.

---

## 14. Debugging command sequence

### HPA and target

```bash
kubectl get hpa -n <namespace>
kubectl describe hpa <hpa> -n <namespace>
kubectl get hpa <hpa> -n <namespace> -o yaml
kubectl get deploy <deployment> -n <namespace> -o yaml
kubectl get deploy <deployment> -n <namespace> -w
```

### Metrics API

```bash
kubectl top pod -n <namespace> --containers
kubectl get apiservice v1beta1.metrics.k8s.io
kubectl describe apiservice v1beta1.metrics.k8s.io
kubectl get --raw '/apis/metrics.k8s.io/v1beta1/namespaces/<namespace>/pods' | jq
```

### Metrics Server

```bash
kubectl get deploy,pods -n kube-system -l k8s-app=metrics-server
kubectl logs -n kube-system deploy/metrics-server --since=30m
kubectl get endpoints -n kube-system metrics-server -o yaml
```

### Requests and limits

```bash
kubectl get deploy <deployment> -n <namespace> -o json | \
  jq '.spec.template.spec.containers[] |
      {name, requests: .resources.requests, limits: .resources.limits}'
```

### Scheduling and readiness

```bash
kubectl get pods -n <namespace> -l app=<label> -o wide
kubectl describe pod <pod> -n <namespace>
kubectl get events -n <namespace> --sort-by=.lastTimestamp
kubectl get endpointslice -n <namespace> \
  -l kubernetes.io/service-name=<service> -o yaml
```

### Conflicting writers

```bash
kubectl get deploy <deployment> -n <namespace> \
  -o jsonpath='{.metadata.managedFields}' | jq
kubectl get events -n <namespace> --sort-by=.lastTimestamp
```

Use managed fields, audit logs, GitOps history, and deployment history to identify who owns `spec.replicas`.

---

## 15. Hands-on incident lab

### Lab objective

Diagnose a service where dashboards show high CPU but the HPA does not deliver sufficient capacity.

### Environment

Deploy:

- a CPU-intensive HTTP service;
- Metrics Server;
- an `autoscaling/v2` HPA;
- a load generator;
- optional Prometheus dashboards;
- a constrained worker-node pool.

### Failure injections

Run each independently:

1. set the CPU request four times higher than expected usage;
2. remove the CPU request from an injected sidecar;
3. set `maxReplicas` below required capacity;
4. restrict scale-up to one pod per minute;
5. block Metrics Server access to one node cohort;
6. add a GitOps loop that reapplies a fixed replica count;
7. constrain nodes so new pods remain Pending;
8. add a three-minute startup and readiness delay;
9. create a proxy sidecar CPU loop while application demand remains flat;
10. generate downstream latency so CPU falls while concurrency explodes.

### Candidate deliverables

For each failure:

- identify the failed transition in the autoscaling chain;
- provide the evidence that proves it;
- choose a reversible mitigation;
- define the permanent correction;
- state which alert would have detected it sooner;
- explain how to prevent the mitigation from creating a second incident.

---

## 16. Principal-level interview answer

A strong spoken answer:

> I would not begin by assuming the HPA controller is broken. I would split the system into observation, calculation, scale mutation, and capacity realization. First I would compare HPA current and desired replicas with the workload's desired and available replicas. Then I would inspect `AbleToScale`, `ScalingActive`, `ScalingLimited`, events, and the exact metrics API values. For CPU utilization, I would verify requests on every relevant container because the HPA uses usage divided by request, not the CPU percentage shown on an arbitrary dashboard.
>
> If the HPA recommends more replicas, I would follow them through scheduling, startup, readiness, EndpointSlice membership, and actual traffic distribution. This catches cases where autoscaling works but the cluster has no capacity, pods warm too slowly, or new replicas never serve traffic. During the incident I would protect critical requests, shed optional work, and manually add capacity only within verified node and downstream headroom.
>
> Long term, I would measure the latency from demand detection to ready serving capacity, remove conflicting ownership of replicas, enforce resource requests, and load-test the full HPA-plus-node-scaling loop. I would keep CPU only when it predicts useful throughput; otherwise I would use concurrency, queue depth, lag, or a carefully designed multi-metric policy. The objective is not merely more pods. It is enough healthy capacity arriving before users consume the error budget.

---

## 17. Interview follow-up questions

### Why can CPU be high while utilization is low?

Because utilization-based HPA metrics divide usage by CPU request. High absolute usage against an even higher request produces low utilization.

### Does the HPA use CPU limits?

Not for `averageUtilization`. CPU requests define the utilization denominator.

### Why can an HPA show more desired replicas while the service remains overloaded?

Because replicas may be Pending, starting, unready, absent from EndpointSlices, unevenly loaded, or blocked by downstream limits.

### What happens when some metrics are missing?

The controller applies conservative assumptions that dampen scaling. Missing metrics can also prevent scale-down when other metrics recommend it.

### Why is `kubectl top` useful but insufficient?

It validates the Metrics API path and raw resource usage, but it does not prove correct requests, target selection, HPA policy, scheduling, readiness, or user-visible capacity.

### When should you use `ContainerResource`?

When one named container represents the intended scaling signal and sidecar CPU should not determine replica count. The sidecars still require independent capacity and saturation monitoring.

### Why not set `minReplicas` to one and rely on fast scale-up?

Because observation, node provisioning, startup, readiness, and load-balancer convergence take time. Baseline capacity must survive that delay and expected failures.

### Why might lowering the target make the outage worse?

It can create a synchronized pod and node surge, exhaust IPs or quotas, overload downstream systems, and increase startup CPU without producing immediate useful capacity.

---

## 18. Design principles to retain

1. **Autoscaling is an end-to-end capacity-delivery system, not an HPA YAML file.**
2. **Resource requests are control-loop inputs. Treat them as production configuration.**
3. **A desired replica is not a scheduled pod; a scheduled pod is not a ready endpoint; a ready endpoint is not proof of useful throughput.**
4. **Measure demand-to-capacity latency.**
5. **Keep enough headroom to survive that latency.**
6. **Use signals with a demonstrated relationship to useful work.**
7. **Give each control field one owner.**
8. **Scale-up and scale-down should be intentionally asymmetric.**
9. **Protect dependencies and users before maximizing replica count.**
10. **Validate the entire control loop with load, failure injection, and rollout testing.**

---

## 19. Official references

- Kubernetes documentation: Horizontal Pod Autoscaling concepts and algorithm.
- Kubernetes API reference: `autoscaling/v2` `HorizontalPodAutoscaler`.
- Kubernetes Metrics Server documentation and requirements.

This chapter describes Kubernetes behavior and a hypothetical Netflix-scale interview scenario. It does not claim knowledge of Netflix's private production architecture.