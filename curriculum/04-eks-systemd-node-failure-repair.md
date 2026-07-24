# Principal Platform Engineering Interview Curriculum

## Chapter 4 — Intermittent systemd Failures on EKS Nodes: Detection, Fencing, and Bounded Repair

> **Original scenario:** An EKS worker node intermittently loses a critical systemd unit such as kubelet, containerd, a CNI component, storage agent, or security daemon. Design detection and automated healing without causing a fleet-wide outage, hiding evidence, violating workload safety, or getting trapped behind graceful-drain assumptions.

---

# 1. Why this question exists

The interviewer is testing whether you can distinguish:

- A process failure from a node failure.
- A transient symptom from a poisoned immutable image.
- Kubernetes node health from application health.
- Graceful voluntary disruption from involuntary infrastructure loss.
- Repair automation from evidence destruction.
- One bad node from a systemic AMI, kernel, CNI, CSI, AZ, or rollout problem.
- A safe restart from an unsafe restart loop.
- A recoverable node from one that must be fenced and replaced.

A weak answer says:

> Add `Restart=always` and have Kubernetes restart the node if it fails.

A senior answer uses systemd restart limits, Node Problem Detector, cordon, drain, and managed node-group replacement.

A Principal-level answer builds a node-health state machine, classifies failure domains, preserves evidence, fences traffic and storage before hard replacement, and places strict fleet-level limits on every automated repair action.

---

# 2. Interview answer first

## 2.1 Ninety-second answer

> I would first identify whether the failed unit is itself the problem or only the first visible symptom of a deeper node failure such as disk exhaustion, OOM, kernel regression, CNI failure, certificate expiry, or filesystem corruption. Detection comes from both the host and the Kubernetes control plane: systemd state and restart counters, journald, kernel logs, kubelet node conditions, EKS node monitoring conditions, workload symptoms, and correlation by AMI, kernel, instance type, AZ, and rollout.
>
> I would use a bounded remediation state machine. A known transient unit failure may receive one controlled restart after evidence capture. Repeated failure, runtime corruption, networking loss, or storage failure causes the node to be tainted or cordoned. If the node and API path are healthy enough, drain gracefully. If the node is terminal or cannot drain, fence it from traffic and storage, verify replacement capacity, and hard-replace it because a dead node cannot honor a PDB.
>
> Automated repair must have maximum parallel repairs, unhealthy-fleet stop thresholds, per-condition wait times, stateful-workload and volume rules, an audit trail, and a global kill switch. EKS automatic node repair can replace or reboot for supported conditions, but I still validate workload tolerance for involuntary node loss and preserve a sample failing node for RCA when the pattern is systemic.
>
> My success criterion is not that systemd eventually shows active. It is that a local node fault is detected quickly, customer capacity remains safe, repair cannot consume the fleet faster than replacement arrives, and the permanent correction is promoted through the immutable AMI or managed add-on pipeline.

## 2.2 Executive summary

> Detect from host and Kubernetes evidence, classify before repairing, use one bounded local restart only for known transients, fence and replace terminal nodes, and stop automation whenever fleet health indicates a systemic problem.

---

# 3. Assumptions and questions

State assumptions:

- Managed EKS control plane.
- Linux worker nodes.
- Managed node groups, Karpenter, EKS Auto Mode, or a combination.
- Critical workloads distributed across zones.
- PodDisruptionBudgets exist but cannot be assumed to protect against dead nodes.
- Nodes are built from immutable AMIs.
- Host access is restricted and evidence collection is automated.

Clarify:

- Which unit failed?
- Does it fail to start, crash, hang, or lose dependencies?
- Is the node still `Ready`?
- Are pods already affected?
- Is the issue one node or a pattern?
- Does the workload use local storage or attached volumes?
- Is the node running quorum members?
- What percentage of fleet can be repaired concurrently?
- Are replacements available in the same or another AZ?
- Is the current rollout still progressing?

Core invariants:

1. Repair automation cannot remove more capacity than the fleet can replace.
2. A failing node is not trusted to report its own health indefinitely.
3. A dead node cannot honor graceful-drain guarantees.
4. Storage fencing precedes replacement when duplicate attachment or writer risk exists.
5. Evidence is preserved before destructive repair when customer safety allows.
6. Permanent fixes are made in images, bootstrap, agents, or workload architecture—not through snowflake SSH changes.

---

# 4. systemd foundations

## 4.1 Unit states

Useful properties:

```bash
systemctl show kubelet \
  -p LoadState \
  -p ActiveState \
  -p SubState \
  -p Result \
  -p NRestarts \
  -p ExecMainStatus \
  -p ActiveEnterTimestamp
```

Interpretation:

- `LoadState`: whether the unit file loaded.
- `ActiveState`: high-level state such as active, inactive, failed.
- `SubState`: more detailed state such as running, exited, dead.
- `Result`: why the most recent activation ended.
- `NRestarts`: restart count managed by systemd.
- `ExecMainStatus`: process exit status.

A unit may appear active while its work is broken. For example, kubelet can run but fail to update node status because DNS, certificates, filesystem, or API connectivity is impaired.

---

## 4.2 Restart policy

Example:

```ini
[Service]
Restart=on-failure
RestartSec=5s

[Unit]
StartLimitIntervalSec=300
StartLimitBurst=3
```

Do not use endless rapid restarts. Restart loops can:

- Flood logs.
- Consume CPU.
- Repeatedly modify network state.
- Hide first-failure evidence.
- Delay node replacement.
- Create API and dependency storms.

A restart policy is local process resilience, not fleet repair.

---

## 4.3 Dependencies and ordering

Inspect:

```bash
systemctl list-dependencies kubelet
systemctl show kubelet -p After -p Before -p Requires -p Wants
systemd-analyze critical-chain kubelet.service
systemd-analyze verify /etc/systemd/system/<unit>.service
```

Common problems:

- Unit starts before network or mount readiness.
- A bootstrap script rewrites configuration after the service starts.
- `Requires=` makes a noncritical dependency fatal.
- A mount or secret file is missing.
- Drop-in overrides conflict with vendor unit files.
- `StartLimitBurst` prevents later restart attempts.

---

# 5. Kubernetes node-health model

## 5.1 Standard conditions

Typical conditions include:

- `Ready`
- `MemoryPressure`
- `DiskPressure`
- `PIDPressure`
- `NetworkUnavailable`

Inspect:

```bash
kubectl get nodes
kubectl describe node <node>
kubectl get node <node> -o json | jq '.status.conditions'
```

A condition is an API-observed statement, not a complete diagnosis.

---

## 5.2 Node Problem Detector

Node Problem Detector can report node problems through:

- `NodeCondition`
- Kubernetes Events
- Metrics

It can monitor:

- System logs.
- Kernel messages.
- Custom plugins.
- Kubelet and runtime health.
- System statistics.

Use NPD to surface a condition, not to build an unbounded destructive repair loop inside the same daemon.

---

## 5.3 EKS node monitoring and automatic repair

The EKS node monitoring agent can surface additional categories such as container runtime, kernel, networking, storage, and accelerated hardware health.

EKS automatic node repair can react with reboot or replacement for supported compute types and conditions. It includes safety behavior such as unhealthy-fleet thresholds; configuration options vary by compute model.

Principal interpretation:

> Managed repair is useful because it integrates node lifecycle with the provider, but it does not remove the need to design workload disruption tolerance, state fencing, capacity headroom, and fleet-level stop conditions.

---

# 6. Failure classification

## 6.1 Kubelet failure

Symptoms:

- Node status becomes stale or `NotReady`.
- Pod status no longer updates.
- New pods do not start.
- Existing containers may continue running temporarily.

Potential causes:

- Kubelet crash or deadlock.
- Certificate expiry.
- API connectivity or DNS failure.
- Invalid config.
- Disk or inode exhaustion.
- Cgroup mismatch.
- Container runtime unavailable.

---

## 6.2 Container runtime failure

Symptoms:

- Kubelet reports runtime unavailable.
- Pod sandbox creation fails.
- Image pulls or container lifecycle operations fail.
- Existing containers may continue depending on failure mode.

Inspect:

```bash
systemctl status containerd
journalctl -u containerd --since '-30 min'
crictl info
crictl ps -a
ctr plugins ls
```

Potential causes:

- Filesystem full.
- Corrupt runtime state.
- Snapshotter failure.
- Cgroup mismatch.
- Version regression.
- Socket permissions.

---

## 6.3 CNI or networking failure

Symptoms:

- New pod sandbox fails.
- Pod-to-pod or service traffic fails.
- Node remains `Ready` while application paths fail.
- DNS fails only on affected nodes.

Inspect:

- CNI agent status and logs.
- Routes and interfaces.
- iptables, IPVS, or eBPF state.
- ENI/IP allocation.
- Conntrack.
- MTU and packet drops.

A CNI restart can itself alter dataplane state. Preserve evidence and understand the agent's recovery model.

---

## 6.4 CSI or storage failure

Symptoms:

- Volume attach/mount failures.
- Filesystem I/O errors.
- Pods stuck terminating or pending.
- Potential duplicate-writer risk.

Do not hard-terminate blindly when storage ownership is uncertain. Fence the node or writer according to the storage system's semantics.

---

## 6.5 Kernel or hardware failure

Evidence:

```bash
dmesg -T
journalctl -k -b
cat /proc/pressure/cpu
cat /proc/pressure/memory
cat /proc/pressure/io
smartctl / nvme diagnostics where supported
EC2 instance status checks and console output
```

Examples:

- Kernel panic.
- Hung task.
- Filesystem corruption.
- NVMe reset.
- ENA driver problem.
- OOM.
- Soft lockup.

A kernel or hardware failure normally justifies replacement rather than process-level repair.

---

# 7. Evidence collection

Collect before reboot or terminate when safe:

```bash
systemctl --failed
systemctl show <unit> -p ActiveState -p SubState -p Result -p NRestarts
journalctl -u <unit> --since '-60 min' --no-pager
journalctl -k --since '-60 min' --no-pager
dmesg -T | tail -500
df -h
df -i
free -m
ps -eo pid,ppid,stat,%cpu,%mem,cmd --sort=-%cpu | head
ss -s
ip addr
ip route
cat /proc/pressure/{cpu,memory,io}
```

Kubernetes evidence:

```bash
kubectl describe node <node>
kubectl get events -A --sort-by=.lastTimestamp
kubectl get pods -A --field-selector spec.nodeName=<node> -o wide
kubectl get pdb -A
kubectl get volumeattachment
```

Cloud evidence:

- Instance status checks.
- ASG or managed node-group activity.
- CloudTrail.
- Launch-template and AMI version.
- Spot interruption notice.
- AZ and instance family.
- EBS attach state.

Automate collection through SSM, a privileged diagnostic DaemonSet, serial console, or node-agent upload so response does not depend on interactive SSH.

---

# 8. Correlation: one node or systemic issue?

Group affected nodes by:

- AMI ID.
- Kernel version.
- Kubernetes version.
- Node group or NodePool.
- Instance type.
- Availability Zone.
- Launch-template version.
- Bootstrap configuration.
- CNI/CSI/runtime version.
- Time since boot.
- Recent rollout wave.

Example query questions:

- Did every failure begin after AMI `ami-123`?
- Is one instance family affected?
- Does the failure occur after exactly N hours?
- Does one AZ show packet loss?
- Did a certificate or secret expire at the same time?
- Does the same unit fail after disk reaches 90%?

A systemic pattern should stop automated replacement before the controller recycles the entire fleet into the same bad image.

---

# 9. Repair state machine

```text
Healthy
  |
  v
Suspect
  | evidence threshold reached
  v
Degraded
  |-- known transient + one retry --> Verify --> Healthy
  |
  +-- repeated/critical --> Cordon/Fence
                            |
                            +-- drain succeeds --> Replace/Reboot
                            |
                            +-- drain impossible and terminal --> Hard replace
```

## 9.1 Suspect

A single log line is not enough for destructive action.

Use:

- Repetition threshold.
- Duration threshold.
- Multiple signals.
- Workload impact.
- Condition-specific logic.

---

## 9.2 One controlled restart

Appropriate only when:

- Failure is known and transient.
- Restart is idempotent.
- Restart does not destroy critical evidence.
- The unit is not repeatedly failing.
- Customer capacity is safe.

```bash
systemctl reset-failed <unit>
systemctl restart <unit>
systemctl is-active <unit>
```

Verification must include functional behavior, not just `active`.

For kubelet:

```bash
kubectl wait node/<node> --for=condition=Ready --timeout=5m
```

For networking, run a connectivity test from a canary pod.

---

## 9.3 Cordon

```bash
kubectl cordon <node>
```

Cordon prevents new scheduling but does not remove existing traffic or pods.

---

## 9.4 Drain

```bash
kubectl drain <node> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --timeout=15m
```

Before drain:

- Verify spare capacity.
- Inspect PDBs.
- Understand local storage.
- Check topology constraints.
- Confirm stateful replica health.
- Confirm replacement provisioning.

Do not override PDBs casually. A drain timeout is diagnostic evidence.

---

## 9.5 Hard replacement

Use when:

- Kubelet/API path is lost.
- Runtime is irrecoverable.
- Node is isolated or poisoned.
- Graceful drain cannot complete.
- Workloads are designed for involuntary node loss.
- Storage and writer fencing are complete.

A PDB protects voluntary disruption; it cannot make a failed physical or virtual node available.

Risks:

- Abrupt connection loss.
- Duplicate job execution.
- Volume attach delay.
- Quorum reduction.
- Data loss for local state.
- Long termination grace not honored.

These risks must be solved in workload architecture, not by assuming every node will drain.

---

# 10. Fencing

Fencing means preventing a failed or ambiguous node from continuing to act as a valid participant.

## 10.1 Traffic fencing

- Remove node from load balancers.
- Mark endpoints unready through control-plane observation.
- Cordon/taint.
- Network-isolate when necessary.

## 10.2 Storage fencing

- Confirm volume detach.
- Use storage-controller fencing.
- Prevent simultaneous writers.
- Verify lease/leader ownership.

## 10.3 Identity fencing

- Revoke node-specific credentials where applicable.
- Prevent a compromised node from retrieving new workload credentials.

Hard termination without fencing can create split-brain or duplicate ownership.

---

# 11. Automation guardrails

Every repair controller needs:

- Maximum concurrent repairs.
- Maximum repairs per interval.
- Unhealthy-fleet threshold.
- AZ-aware limits.
- Node-group-aware limits.
- Per-condition waiting period.
- Exclusion for debugging/quarantine nodes.
- State and storage rules.
- Global disable switch.
- Audit event.
- Notification.
- Rollout correlation.

Example policy:

```text
Normal:
  max 1 node per AZ
  max 3 nodes cluster-wide

Stop automatically when:
  >10% of node group unhealthy
  >1 AZ affected
  replacement provisioning fails
  same AMI shows repeated failure
  Tier-0 spare capacity < required threshold
```

Do not let the repair mechanism become the largest source of disruption.

---

# 12. PDBs and their limits

PodDisruptionBudgets constrain voluntary evictions.

They do not prevent:

- Node crash.
- EC2 termination.
- Kernel panic.
- Network partition.
- Spot interruption beyond available notice.
- Storage failure.

A good design combines:

- Replicas.
- Topology spread.
- Anti-affinity.
- PDBs.
- Capacity headroom.
- Graceful shutdown.
- Involuntary-failure tolerance.

Dangerous statement:

> We cannot terminate the dead node because the PDB blocks it.

Correct framing:

> If the node is truly dead, the workload has already lost that capacity. We fence the failed instance and restore replicas elsewhere; the PDB informs voluntary maintenance but cannot resurrect the node.

---

# 13. Node lifecycle architectures

## Managed node groups

Benefits:

- Integrated replacement and update lifecycle.
- Launch-template versioning.
- EKS repair support.

## Karpenter

Benefits:

- Flexible provisioning.
- NodeClaim lifecycle.
- Rapid replacement and instance diversification.

Risks:

- Disruption settings and consolidation must align with workload safety.
- Cloud capacity or subnet limits can block replacement.

## EKS Auto Mode

Provides managed node and integrated capabilities, but does not remove application-level disruption design.

Principal rule:

> Choose a node lifecycle manager, but maintain one clear owner for replacement. Competing controllers can create oscillation.

---

# 14. Incident response example

## Scenario

Twenty minutes after an AMI rollout, containerd fails on 4% of nodes. Automated restarts temporarily recover it, then it fails again. Pending pods rise.

### Stop the blast radius

- Pause node rollout.
- Disable broad auto-repair if replacement would launch the same AMI.
- Freeze unrelated node changes.
- Verify customer capacity.

### Bound

- Same AMI?
- Same instance family?
- Same AZ?
- Same disk size?
- Same runtime version?

### Preserve evidence

Collect runtime logs, kernel logs, filesystem usage, snapshotter state, PSI, AMI metadata, and instance console output.

### Mitigate

- Cordon affected nodes.
- Replace with last-known-good launch-template version.
- Maintain concurrency limits.
- Shift noncritical workloads if capacity is low.

### Root cause example

A runtime configuration change causes snapshotter metadata growth until the filesystem fills.

Trigger:

- New AMI rollout.

Systemic cause:

- Qualification did not include realistic image churn and disk-pressure soak.

Permanent correction:

- Fix image.
- Add disk/inode tests.
- Add runtime-state growth alert.
- Canary longer.
- Block promotion on repeated container-runtime conditions.

---

# 15. Observability

Host metrics:

- Unit active/failed state.
- Restart count.
- Exit status.
- Journal error rate.
- Disk/inode usage.
- PSI.
- OOM and kernel warnings.
- Runtime operation latency.

Kubernetes metrics:

- Node conditions.
- Time `NotReady`.
- Pending pods.
- Eviction and drain duration.
- Pod rescheduling time.
- PDB-blocked eviction.

Repair metrics:

- Repair attempts by reason.
- Success/failure.
- Time to fence.
- Time to replacement ready.
- Concurrent repairs.
- Automation stop events.

Product metrics:

- Capacity margin.
- Request error/latency by node group and AZ.
- Stateful quorum health.

---

# 16. Dangerous answers

## “Restart the unit forever.”

This hides systemic defects and can create repeated disruption.

## “Use the Descheduler to repair nodes.”

The Descheduler moves pods according to policy; it is not a general poisoned-node repair controller.

## “Always drain before terminate.”

Graceful drain is preferred when possible, but terminal nodes may not be able to participate. Fencing and hard replacement must exist.

## “PDBs guarantee availability.”

They cover voluntary eviction, not all failure modes.

## “Terminate all unhealthy nodes immediately.”

Without rate limits and replacement capacity, repair can create a fleet outage.

## “SSH and patch the node.”

Use interactive debugging only to collect evidence or validate a hypothesis. Durable changes belong in immutable automation.

---

# 17. Acceptance criteria

- Known transient unit failure recovers within target after one restart.
- Repeated failure transitions to replacement, not an infinite loop.
- Evidence is uploaded before destructive action when safe.
- Replacement node reaches `Ready` within the node provisioning SLO.
- Auto-repair never exceeds configured fleet and AZ limits.
- A bad AMI correlation stops rollout and repair automatically.
- Tier-0 workload remains within SLO during one node and one-AZ experiments.
- Stateful workload passes involuntary-node-loss test.
- Storage fencing is proven.
- Last-known-good AMI rollback is automated.

---

# 18. Adversarial follow-ups

### Why not just reboot?

A reboot can recover some kernel or service states, but it destroys volatile evidence and may repeat indefinitely. I classify the failure and use reboot only for known repairable conditions.

### What if drain is blocked by a PDB?

If the node is healthy enough, I wait or create capacity. If it is terminal, the pod is already unavailable; I fence and replace while restoring replicas elsewhere.

### What if replacement capacity is unavailable?

Stop additional repairs, preserve existing capacity, relax noncritical scheduling, use diversified instance types or another AZ, and escalate quota/capacity. Do not consume the remaining fleet.

### How do you preserve a failing node?

Quarantine one representative node, remove it from traffic, prevent scheduling, upload evidence, and stop it from acting as a storage writer.

### When is a process restart safe?

When the failure is understood, restart is idempotent, evidence is preserved, and repeated failure moves to a stronger state.

---

# 19. Hands-on lab

1. Create a disposable EKS node group.
2. Deploy a replicated workload with PDB and topology spread.
3. Install Node Problem Detector or the EKS node monitoring agent.
4. Create a test systemd unit that fails intermittently.
5. Observe unit state, events, and node conditions.
6. Implement one-restart remediation.
7. Force repeated failure and verify cordon.
8. Test graceful drain.
9. Simulate kubelet loss and perform hard replacement in the lab.
10. Verify rescheduling, volume behavior, and repair limits.
11. Launch multiple failures and prove the fleet circuit breaker stops automation.

Lab report:

- Which signal was authoritative?
- How long until detection?
- What evidence survived?
- Did the PDB affect drain?
- What happened during involuntary loss?
- Did replacement capacity arrive first?
- Did automation stop at its limit?

---

# 20. Memorization card

```text
Unit failure is not automatically node failure
 -> inspect systemd, journal, kernel, runtime, disk, network
 -> correlate by AMI/kernel/AZ/instance type
 -> one bounded restart only for known transient
 -> repeated or critical failure: cordon/fence
 -> drain when possible
 -> hard replace terminal node after traffic/storage fencing
 -> PDB protects voluntary disruption, not dead hardware
 -> rate-limit repair and stop on systemic pattern
 -> permanent fix belongs in immutable image or managed component
```

Final Principal sentence:

> I treat node repair as a safety-critical control loop: it must restore capacity faster than it removes it, preserve enough evidence to eliminate the defect, and stop automatically when the pattern indicates the fleet—not the individual node—is unhealthy.

---

# 21. Official primary references

- EKS node health and node monitoring agent:  
  https://docs.aws.amazon.com/eks/latest/userguide/node-health.html
- EKS automatic node repair:  
  https://docs.aws.amazon.com/eks/latest/userguide/node-repair.html
- Kubernetes Node Problem Detector:  
  https://kubernetes.io/docs/tasks/debug/debug-cluster/monitor-node-health/
- Kubernetes node-pressure eviction:  
  https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/
- Kubernetes disruptions and PDBs:  
  https://kubernetes.io/docs/concepts/workloads/pods/disruptions/
- systemd.service manual:  
  https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html
