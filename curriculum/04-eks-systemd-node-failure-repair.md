# Netflix Adapter — EKS Node Failure, Fencing, and Bounded Repair

> **Original scenario:** An EKS worker node intermittently loses a critical systemd unit such as kubelet, containerd, a CNI component, storage agent, or security daemon. Design detection and automated healing without causing a fleet-wide outage, hiding evidence, violating workload safety, or getting trapped behind graceful-drain assumptions.

This file is a thin Netflix/media-platform adapter. Reusable Linux systemd, kubelet/runtime, node lifecycle, fencing, drain, repair, node-image, Kubernetes probe, storage, and chaos theory is canonical in the shared handbook.

## Canonical prerequisites

- [Kubernetes node failure, fencing, and repair](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/core/kubernetes/node-lifecycle/failure-fencing-repair.md)
- [Kubernetes runtime restart and OOM debugging](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/core/kubernetes/runtime-debugging.md)
- [Kubernetes node-image qualification, promotion, and rollback](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/core/kubernetes/node-images/qualification-promotion-rollback.md)
- [Linux architecture, boot, systemd, and syscalls](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/core/linux/01-architecture-boot-syscalls.md)
- [Kubernetes probes, startup, shutdown, and traffic drain](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/core/kubernetes/workload-lifecycle/probes-startup-shutdown-drain.md)
- [Kubernetes CSI and stateful recovery](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/core/kubernetes/storage/csi-stateful-recovery.md)
- [Chaos engineering and game-day governance](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/core/reliability/chaos-engineering-game-days.md)

## Netflix-shaped concerns

The media-platform scenario adds:

- playback-critical services spread across many cells and node pools;
- high connection counts and long-lived streams;
- large node fleets where unsafe automation can amplify one image or agent defect;
- strict requirements to preserve traffic capacity during repair;
- mixed stateless, cache, queue-consumer, and stateful-writer workloads;
- regional release rings and immutable node images;
- the need to distinguish one bad node from a systemic kernel, image, CNI, runtime, or configuration release.

## Ninety-second answer

> I would model node repair as a guarded state machine, not a restart script. First I establish whether the fault is node-local or systemic by correlating systemd unit, kernel, node image, instance type, zone, cluster, and failure signature. I preserve journal, kubelet, runtime, kernel, CNI, storage, and workload evidence before destructive action.
>
> A single transient local failure may receive one bounded restart and verification. Repeated failure moves the node toward cordon, traffic removal, fencing, and replacement with a known-good image. A storage writer, singleton, or partitioned node must be fenced before replacement or failover. Drain is used only when the node and control path are healthy enough and disruption budgets, topology, and connection drain make it safe.
>
> Repair concurrency is limited per cluster and zone, minimum healthy playback capacity is protected, and a fleet circuit breaker stops automation when the same image or signature appears across multiple zones or reaches a threshold. New capacity must become Ready and serve synthetic playback traffic before the next replacement begins.
>
> Recovery is proven through node and workload SLIs, endpoint and gateway state, playback-start and rebuffer metrics, and replacement convergence. The permanent fix is a new qualified immutable node image or platform release, followed by a game day reproducing the failure and demonstrating bounded repair.

## State machine

```text
HEALTHY
  -> SUSPECT
  -> EVIDENCE_CAPTURED
  -> one bounded restart and VERIFYING
       | success -> HEALTHY
       | repeat  -> DEGRADED
  -> CORDONED
  -> TRAFFIC_FENCED
  -> STORAGE / IDENTITY FENCED when required
  -> DRAINING when safe
  -> REPLACING
  -> REPLACEMENT_READY
  -> workload and playback verification
  -> RETIRED
```

A systemic pattern transitions the fleet controller to:

```text
AUTOMATION_DISABLED
  -> stop rollout
  -> preserve old capacity
  -> isolate image/revision/signature
  -> human incident command
  -> known-good rollback or replacement plan
```

## Detection and evidence

Correlate:

- systemd unit state, restart count, and exit reason;
- journal and previous-boot logs;
- kubelet heartbeat and Node conditions;
- container runtime, CNI, CSI, security, and observability agents;
- kernel warnings, OOM, disk, inode, PID, and network pressure;
- node image, bootstrap version, kernel, runtime, instance type, architecture, zone, and cluster;
- pod sandbox, DNS, Service, storage, and image-pull failures;
- affected workload class and playback/user cohort;
- recent image, configuration, agent, or policy rollout.

Useful entry points:

```bash
systemctl status kubelet containerd
journalctl -u kubelet -u containerd --since '-30 min'
journalctl -b -p warning
systemctl --failed
kubectl describe node <node>
kubectl get pods -A -o wide --field-selector spec.nodeName=<node>
kubectl get events -A --sort-by=.lastTimestamp
```

## Repair policy

### One bounded restart

Allowed only when:

- failure is local and transient;
- restarting cannot violate writer, safety, or identity invariants;
- enough healthy capacity exists;
- no systemic signature is active;
- evidence is already preserved;
- success is verified through mechanism and workload signals.

### Cordon and drain

Use when the node is responsive and workloads can terminate safely. Verify:

- PodDisruptionBudgets;
- topology and replacement capacity;
- long-lived connection drain;
- queue-worker lease/redelivery semantics;
- local storage and `emptyDir` implications;
- stateful writer ownership;
- external load-balancer and EndpointSlice propagation.

### Fence before replacement

Required when:

- node is partitioned or untrusted;
- storage writer may still commit;
- identity or credentials may still be active;
- traffic may still reach the node;
- graceful drain cannot be trusted.

Cordon is not fencing. NotReady is not fencing.

## Fleet safety limits

Enforce:

- maximum concurrent repairs per cluster;
- maximum concurrent repairs per zone or cell;
- minimum healthy capacity percentage;
- one replacement Ready before the next destructive transition;
- protected playback and control-plane capacity;
- rollout ring boundaries;
- global disable switch;
- circuit breaker by image failure percentage;
- circuit breaker by repeated signature across zones;
- repair and rollback audit trail.

## Netflix-specific recovery proof

A replacement node is not accepted merely because it is `Ready`.

Verify:

- kubelet, runtime, CNI, CSI, security, and telemetry agents;
- pod sandbox and image pull;
- Service DNS and endpoint connectivity;
- policy and identity convergence;
- storage attach and mount where applicable;
- synthetic manifest, entitlement, DRM, metadata, and segment requests;
- playback-start success and latency;
- rebuffer and fatal playback-error cohorts;
- no concentration of replicas in one zone or node-image cohort.

## Failure modes

### Same image fails across zones

Stop automation and image promotion. Preserve old capacity. Treat it as a release incident, qualify a known-good replacement, and avoid replacing the fleet into the same defect.

### CNI or runtime loss leaves pods apparently Running

Do not trust object state alone. Test pod sandbox creation, Service/DNS paths, endpoint reachability, and new connection establishment.

### Node is partitioned but still serving or writing

Fence traffic, storage, and identity before replacement. Avoid dual writer and duplicate queue-consumer behavior.

### Drain stalls

Classify the blocker: PDB, finalizer, long request, connection, local data, CSI, unresponsive kubelet, or application shutdown. If graceful control is not reliable, switch to the documented fenced replacement path.

## SLOs

Track:

- node-fault detection time;
- evidence-capture success;
- repair decision time;
- one-restart success versus repeat-failure rate;
- cordon-to-traffic-removal time;
- drain duration and forced-termination rate;
- fencing confirmation time;
- replacement launch-to-Ready and launch-to-serving time;
- repair concurrency by zone and cluster;
- systemic circuit-breaker activations;
- playback SLI impact during repair;
- image-cohort and failure-signature recurrence.

## Adversarial follow-ups

### Why not restart kubelet automatically every time?

Repeated restarts hide evidence, can amplify a systemic release defect, and may leave networking, storage, runtime, or workload state unsafe. One bounded restart is a hypothesis test, not the whole repair strategy.

### Why not always drain?

Drain assumes a responsive control path and cooperative workloads. A partitioned or failed node may require fencing and replacement first, especially for writers or stale traffic paths.

### How do you prevent automation from destroying an availability zone?

Per-zone concurrency, minimum-capacity guards, one-replacement-ready gates, cell boundaries, rollout rings, and a systemic-failure circuit breaker.

### What if the node reports Ready but playback still fails?

Node Ready reflects a limited control-plane signal. Verify CNI, DNS, Services, proxy, storage, application, and synthetic playback paths.

### What is the permanent fix?

Correct the image, kernel, runtime, agent, configuration, or dependency; qualify it through canary and failure tests; roll it out by ring; and re-run the node-failure game day.

## Practical drill

Run:

- [Kubernetes node-repair state-machine lab](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/tree/main/labs/kubernetes/01-node-repair-state-machine)
- [Disposable Kind scheduling, DNS, probe, and drain conformance](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/tree/main/labs/kubernetes/02-kind-conformance)

Then explain how the state-machine guards change for playback frontends, cache nodes, queue consumers, and storage writers.

## Personal-story bridge

Use a truthful example involving Linux/systemd failure, Kubernetes node repair, immutable image rollout, capacity protection, stateful fencing, or stopping automation after recognizing a systemic pattern. Separate actual production evidence from this hypothetical streaming-platform design.
