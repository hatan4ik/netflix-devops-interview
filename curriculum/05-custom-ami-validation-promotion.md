# Principal Platform Engineering Interview Curriculum

## Chapter 5 — Custom AMI Validation, Qualification, Promotion, and Rollback

> **Original scenario:** Your platform team builds custom Amazon Machine Images for Kubernetes worker nodes. Design a production-grade validation and promotion process that proves a new AMI is safe before broad rollout. Explain what must be tested, how promotion works, how you detect hidden regressions, and how you roll back without creating a fleet-wide incident.

---

# 1. What the interviewer is testing

This question is not really about Packer syntax.

The interviewer wants to know whether you understand that a node image is a production artifact containing an operating system, kernel, container runtime, kubelet, networking dependencies, storage drivers, security agents, bootstrap logic, certificates, package repositories, and configuration defaults.

A weak answer says:

> Build the AMI with Packer, launch one EC2 instance, run a smoke test, and deploy it.

A strong Staff or Principal answer treats the AMI as a versioned platform release and explains:

- Deterministic image construction.
- Supply-chain integrity and provenance.
- Static validation before boot.
- Boot and bootstrap validation.
- Kubernetes join and lifecycle tests.
- CNI, CSI, DNS, service discovery, and load-balancer tests.
- Runtime, kernel, cgroup, and systemd compatibility.
- Performance and resource-regression testing.
- Security validation and vulnerability handling.
- Canary node groups and workload-safe draining.
- Promotion gates based on customer and platform SLIs.
- Rollback and forward-fix strategies.
- Failure-domain control.

The central principle is:

> A node image is safe only after it proves that workloads can be scheduled, started, networked, observed, drained, upgraded, and recovered under realistic failure conditions.

---

# 2. Ninety-second interview answer

> I would treat every AMI as an immutable, signed platform release with explicit provenance, automated qualification, staged promotion, and a tested rollback path. The pipeline starts from an approved base image pinned by image ID and package versions. Packer or EC2 Image Builder creates the artifact, records the source commit, build manifest, SBOM, package inventory, kernel configuration, and cryptographic digest, and then publishes it only into a quarantine account or nonproduction catalog.
>
> Validation happens in layers. First I run static checks for required packages, users, permissions, sysctl settings, services, certificates, container-runtime configuration, kubelet flags, kernel modules, cgroup mode, and prohibited software. Then I boot representative instance types in every supported architecture and Availability Zone, verify cloud-init and systemd ordering, join an ephemeral EKS cluster, and run Kubernetes conformance plus platform-specific tests for CNI, DNS, CSI, image pulls, secrets, logging, metrics, service load balancing, ingress, pod disruption, node pressure, reboot, drain, and replacement.
>
> I also compare performance against the current production image: boot time, node-ready latency, pod-start latency, network throughput, connection churn, disk latency, CPU overhead, memory baseline, conntrack behavior, and maximum pod density. A passing image is promoted through dev, staging, shadow, and production canary node groups. Real workloads move gradually using taints, tolerations, node affinity, and disruption budgets. Promotion stops automatically on customer-SLI regression, node-not-ready growth, CNI or CSI errors, image-pull failures, kernel warnings, OOM changes, security-agent failure, or abnormal drain and replacement time.
>
> Rollback means creating capacity from the previously approved AMI, cordoning and draining the bad nodes, and replacing them within bounded disruption budgets. I would never mutate failed nodes in place as the primary recovery strategy. The image catalog maintains an immutable promotion history, and each production rollout includes a rollback rehearsal, stop conditions, ownership, and maximum blast radius.

---

# 3. Assumptions to state

Use an opening such as:

> I will assume EKS managed or self-managed node groups across multiple Availability Zones, more than one instance family, Linux worker nodes, a CNI and CSI plugin, DaemonSet-based security and observability agents, and workloads with disruption budgets. I will also assume the platform has a current known-good image that can be restored quickly.

Clarify:

- EKS managed node groups, Karpenter, Auto Scaling Groups, or a combination.
- CPU architectures: x86_64, arm64, or both.
- GPU, Inferentia, local-NVMe, or high-network-throughput nodes.
- Bottlerocket, Amazon Linux, Ubuntu, RHEL, or another distribution.
- Container runtime and version.
- cgroup v1 or v2.
- Current CNI and kube-proxy mode.
- Encrypted root and data volumes.
- FIPS or compliance requirements.
- Whether nodes have internet egress.
- Required maximum node-replacement time.
- Stateful workloads and local-storage usage.
- Maintenance windows and disruption tolerance.

---

# 4. Why AMIs are high-risk platform artifacts

A node image affects every pod placed on that node.

One small image change can produce a broad outage:

- A kernel update changes network-driver behavior.
- A container-runtime update changes cgroup handling.
- A kubelet flag prevents node registration.
- A missing CA certificate breaks image pulls or API access.
- A systemd dependency delays the CNI agent.
- A sysctl change exhausts conntrack.
- A storage package update prevents volume attachment.
- A security hardening rule blocks required kernel modules.
- A logging agent consumes excessive CPU.
- A bootstrap script succeeds partially and leaves a node falsely healthy.
- A new package repository becomes unavailable during scale-out.

The risk is multiplicative because autoscaling may launch hundreds or thousands of identical broken nodes during an incident.

Therefore the image pipeline itself must be designed as a safety system.

---

# 5. Artifact model and immutability

Every image should have a machine-readable identity.

Example:

```yaml
image:
  name: eks-worker
  version: 2026.07.26-3
  source_ami: ami-0123456789abcdef0
  git_commit: 3b1f1a9
  kernel: 6.1.142
  containerd: 2.0.5
  kubelet: 1.34.1
  architecture: x86_64
  build_pipeline: platform-ami-prod
  sbom_digest: sha256:...
  image_digest: sha256:...
  promotion_state: candidate
```

Do not use a mutable label such as `latest` as the deployment source of truth.

Use:

- Immutable AMI IDs.
- Versioned launch templates.
- Signed build manifests.
- SBOMs.
- Package locks or snapshots.
- Recorded kernel configuration.
- Recorded bootstrap version.
- Recorded test results.
- Promotion metadata.

A human-readable name is useful, but the AMI ID and manifest digest are authoritative.

---

# 6. Deterministic image construction

## 6.1 Pin the base image

Do not resolve the base image dynamically during a production promotion.

Bad:

```hcl
source_ami_filter {
  filters = {
    name = "amazon-eks-node-*"
  }
  most_recent = true
}
```

This makes rebuilds non-reproducible.

Better:

```hcl
source_ami = var.approved_source_ami_id
```

A separate controlled process discovers, reviews, and approves a new base image.

## 6.2 Pin packages and repositories

Risks include:

- Repository metadata changes during a build.
- A package disappears.
- A transitive dependency changes.
- A mirror serves different content.

Use repository snapshots, version pins, checksums, and package manifests where practical.

## 6.3 Minimize first-boot mutation

The more software installed at boot, the less confidence the AMI qualification provides.

Prefer baking stable components into the image:

- Container runtime.
- Required kernel modules.
- Base monitoring dependencies.
- Security-agent prerequisites.
- CA bundles.
- Filesystem tooling.
- SSM agent.

Use bootstrap for environment-specific identity and configuration, not for reconstructing the operating system.

---

# 7. Build pipeline stages

```text
approved source image
        |
        v
image build
        |
        v
static inspection
        |
        v
quarantine AMI
        |
        v
boot matrix tests
        |
        v
ephemeral Kubernetes qualification
        |
        v
performance and resilience tests
        |
        v
security approval
        |
        v
dev promotion
        |
        v
staging promotion
        |
        v
production canary
        |
        v
progressive fleet rollout
        |
        v
approved production image
```

Each arrow is a recorded gate, not an informal handoff.

---

# 8. Static validation before boot

Static validation should inspect the mounted image or a launched builder instance before promotion.

Validate:

- Expected OS release.
- Kernel and initramfs.
- Installed package inventory.
- Package signatures.
- Required binaries and versions.
- Disabled or prohibited services.
- File ownership and permissions.
- SSH configuration.
- sudoers configuration.
- SELinux or AppArmor state.
- sysctl settings.
- limits.conf and systemd limits.
- containerd configuration.
- kubelet configuration and bootstrap scripts.
- CA trust store.
- Time synchronization.
- Log rotation.
- SSM or break-glass access.
- Required kernel modules.
- Blacklisted modules.
- FIPS state where required.
- Root-volume encryption.
- No embedded credentials.

Example checks:

```bash
set -euo pipefail

test "$(stat -c '%U:%G:%a' /etc/kubernetes/bootstrap.sh)" = "root:root:700"
systemctl is-enabled containerd
systemctl is-enabled kubelet
containerd --version
kubelet --version
sysctl net.ipv4.ip_forward
sysctl net.netfilter.nf_conntrack_max
find / -xdev -type f \( -name '*.pem' -o -name '*secret*' \) -print
```

Static checks are necessary but insufficient because many failures occur only during boot or under workload.

---

# 9. Boot qualification matrix

Test every supported combination that can change behavior:

| Dimension | Examples |
|---|---|
| Architecture | x86_64, arm64 |
| Instance family | general, compute, memory, network, GPU |
| Storage | EBS-only, local NVMe |
| Availability Zone | each production AZ |
| Network mode | IPv4, dual-stack where used |
| Capacity type | on-demand, Spot |
| Kernel branch | standard, low-latency, FIPS |
| Cluster version | current and next supported Kubernetes version |

Validate:

- EC2 reaches running state.
- Console output has no boot failure.
- cloud-init completes.
- Required systemd units start in the correct order.
- Time synchronization is healthy.
- IMDS configuration matches policy.
- Root filesystem expands correctly.
- Local disks are discovered and formatted safely.
- Network interfaces and ENIs initialize.
- SSM access works where required.
- Reboot returns to a healthy state.
- Shutdown terminates cleanly.

Important metrics:

- Instance launch to SSH/SSM readiness.
- Launch to kubelet start.
- Launch to Kubernetes Node object.
- Launch to `Ready=True`.
- Launch to first schedulable pod.

Use percentiles, not only averages.

---

# 10. Kubernetes node-join validation

A node is not qualified merely because it becomes `Ready`.

Test the complete registration path:

```text
EC2 launch
  -> bootstrap obtains cluster details
  -> kubelet starts
  -> credentials authenticate
  -> node object created
  -> CNI initializes
  -> CSI and required DaemonSets initialize
  -> node becomes Ready
  -> scheduler places workload
  -> workload reaches application readiness
```

Validate:

- Correct node labels and taints.
- Correct region, AZ, architecture, and instance-type labels.
- Correct allocatable CPU, memory, pods, ephemeral storage, and devices.
- Expected max-pod calculation.
- Kubelet certificate bootstrap and rotation.
- Container-runtime endpoint.
- Image credential provider behavior.
- Node lease updates.
- Node status under API latency.
- Graceful node shutdown.
- Eviction thresholds.
- Reserved resources.
- Log and image garbage collection.

A frequent hidden defect is a node that joins but reports incorrect capacity or fails after certificate rotation.

---

# 11. Platform integration tests

## 11.1 CNI and networking

Test:

- Pod-to-pod traffic on the same node.
- Pod-to-pod traffic across nodes and AZs.
- Pod-to-Service traffic.
- Pod-to-external traffic.
- Ingress traffic.
- NetworkPolicy enforcement.
- DNS resolution.
- Connection tracking under churn.
- MTU and fragmentation behavior.
- IPv6 where used.
- Source-IP preservation where required.
- Pod IP allocation and release.
- Node replacement under IP pressure.

Measure:

- Throughput.
- Packets per second.
- Connection setup latency.
- Retransmits.
- Drops.
- conntrack occupancy.
- CNI allocation latency.

## 11.2 DNS

Test:

- Cluster service names.
- Headless services.
- External names.
- Negative caching.
- Search-domain expansion.
- High query concurrency.
- DNS during CoreDNS restart.
- DNS after node reboot.

## 11.3 Storage

Test:

- EBS volume attach, mount, unmount, and detach.
- Filesystem expansion.
- Volume snapshot restore.
- Stateful pod rescheduling.
- Multi-AZ failure behavior.
- Ephemeral-storage pressure.
- Local NVMe initialization where used.
- KMS-encrypted volume access.

## 11.4 Image pulls and registries

Test:

- Private ECR pulls.
- Cross-account pulls.
- Large image pulls.
- Credential refresh.
- Registry throttling.
- Pull after node reboot.
- Image garbage collection.

## 11.5 Secrets and identity

Test:

- IRSA or workload identity.
- Secrets Store CSI.
- KMS decrypt permissions.
- Credential rotation.
- No unintended access through node instance profile.

## 11.6 Observability and security DaemonSets

Validate that required agents start and remain healthy:

- Metrics agent.
- Log collector.
- OpenTelemetry collector.
- Security/runtime agent.
- CNI agent.
- CSI node plugin.
- Node-local DNS where used.

A node should not become generally schedulable until critical node services are ready.

---

# 12. Workload lifecycle tests

Use representative workloads, not only `nginx`.

Include:

- Small stateless HTTP service.
- High-connection-rate service.
- CPU-bound service.
- Memory-sensitive service.
- Stateful workload.
- Large container image.
- Service-mesh-injected pod.
- Security-restricted pod.
- Workload with termination hooks.
- Workload using persistent volumes.

Test:

- Scheduling.
- Startup.
- Readiness.
- Liveness behavior.
- Graceful termination.
- Eviction.
- Drain.
- Rescheduling.
- PDB behavior.
- Topology spread.
- HPA and cluster autoscaling interaction.
- Pod restart after container-runtime restart.
- Node reboot.

---

# 13. Performance regression testing

Compare the candidate against the current approved image using the same instance type and workload.

Key metrics:

- Boot-to-ready latency.
- Pod-start latency.
- Container image unpack time.
- CPU idle baseline.
- Memory baseline.
- Kernel slab usage.
- Network throughput.
- Network p99 latency.
- Connection setup rate.
- Disk IOPS and latency.
- Filesystem throughput.
- Maximum stable pod density.
- DaemonSet resource overhead.
- Drain duration.
- Reboot recovery time.

Define explicit budgets, for example:

```text
boot-to-ready p95 regression      <= 10%
pod-start p95 regression          <= 10%
network p99 regression            <= 5%
idle memory increase              <= 150 MiB
node DaemonSet CPU increase       <= 5%
packet loss                       = 0 outside test fault injection
```

The exact thresholds depend on the platform, but promotion must be governed by measurable limits.

---

# 14. Failure and resilience tests

A candidate image must survive more than the happy path.

Inject:

- API server latency and temporary unavailability.
- DNS interruption.
- Registry throttling.
- CNI restart.
- Container-runtime restart.
- Kubelet restart.
- Node reboot.
- Disk pressure.
- Memory pressure.
- PID pressure.
- ENI or pod-IP pressure.
- Temporary loss of external egress.
- Security-agent restart.
- Log-agent backpressure.
- Spot interruption.

Observe whether:

- The node recovers automatically.
- Workloads remain within SLO.
- The node reports accurate conditions.
- Eviction occurs in the intended order.
- Alerts fire with actionable context.
- Repair automation avoids loops.
- Repeated restart does not corrupt state.

---

# 15. Security qualification

Security validation includes both vulnerabilities and configuration.

Check:

- CVE scan with documented exception policy.
- SBOM generation.
- Package and repository signatures.
- Kernel lockdown and module policy.
- SELinux/AppArmor state.
- seccomp defaults.
- IMDSv2 enforcement.
- SSH exposure.
- Root login policy.
- Unused ports and services.
- File permissions.
- Audit logging.
- Credential leakage.
- Image sharing permissions.
- KMS encryption.
- SSM controls.
- CIS or organizational benchmark.

Do not automatically reject every CVE without context. Evaluate:

- Is the vulnerable component installed?
- Is the vulnerable path reachable?
- Is a fix available?
- Does the fix create greater production risk?
- Is compensating control available?
- What is the remediation deadline?

The decision must be recorded and time-bounded.

---

# 16. Promotion model

Use immutable promotion rather than rebuilding at every stage.

```text
candidate AMI ID
     |
     +--> dev approved
     |
     +--> staging approved
     |
     +--> production canary approved
     |
     +--> production approved
```

Promote the same artifact across accounts and regions when possible. Rebuilding can introduce differences.

Promotion metadata should include:

- AMI ID per region.
- Source digest.
- Test suite version.
- Test results.
- Approvers.
- Timestamps.
- Exceptions.
- Rollback image.
- Supported Kubernetes versions.
- Supported instance families.

---

# 17. Production canary rollout

Create a dedicated canary node group or Karpenter node class.

Use:

- Small fixed maximum capacity.
- One or more nodes per AZ.
- Explicit labels and taints.
- Selected representative workloads.
- Strict disruption budgets.
- Separate dashboards and alerts.
- Automatic stop conditions.

Example:

```yaml
metadata:
  labels:
    platform.example.com/ami-channel: canary
spec:
  taints:
    - key: platform.example.com/ami-canary
      value: "true"
      effect: NoSchedule
```

Only canary workloads receive the corresponding toleration initially.

Expand progressively:

```text
1% nodes
 -> 5%
 -> one low-risk cell
 -> 25%
 -> 50%
 -> 100%
```

Progression should depend on elapsed observation time and traffic volume, not only node count.

---

# 18. Stop conditions

Automatically halt promotion when any of the following exceeds its budget:

- Node join failure.
- Node-ready latency.
- `NotReady` rate.
- Kubelet restart rate.
- Container-runtime restart rate.
- CNI or CSI initialization errors.
- Pod sandbox creation failures.
- Image-pull errors.
- DNS errors.
- Packet drops or retransmits.
- Volume attachment failures.
- Kernel warnings, panics, or soft lockups.
- OOM kills.
- Unexpected eviction.
- Security-agent absence.
- Log or metric gaps.
- Customer latency or error-rate regression.
- Drain time exceeding budget.
- Replacement capacity failing to arrive.

A Principal-level answer emphasizes that **customer SLIs outrank infrastructure self-reported health**.

---

# 19. Rollback design

Rollback should be prepared before rollout.

## 19.1 Preferred rollback

1. Stop further provisioning from the candidate image.
2. Set the previous launch-template or node-class version as default.
3. Create sufficient replacement capacity from the known-good image.
4. Confirm replacement nodes are healthy and schedulable.
5. Cordon candidate nodes.
6. Drain them within PDB and capacity limits.
7. Terminate and replace them.
8. Verify customer and platform SLIs recover.

## 19.2 Why not patch nodes in place

In-place repair creates drift and weakens auditability.

It may be useful as an emergency containment action, but the steady-state correction is replacement with a known artifact.

## 19.3 Rollback risks

- Old image no longer supports the upgraded control plane.
- Old image has expired certificates or package dependencies.
- Capacity is unavailable for replacement.
- PDBs prevent drain.
- Local-storage workloads cannot move.
- Autoscaler repeatedly launches the bad image.
- Multiple launch-template versions are active unintentionally.

Therefore retain and continuously revalidate at least one rollback image compatible with the current cluster version.

---

# 20. Debugging workflow for a failed candidate

Use a layered workflow.

## Layer 1: Build provenance

- Confirm exact AMI ID.
- Confirm source AMI.
- Confirm build commit and manifest.
- Compare package inventory with known-good image.
- Review build logs and exceptions.

## Layer 2: Boot

- EC2 console output.
- Cloud-init logs.
- `journalctl -b`.
- Failed systemd units.
- Kernel command line.
- Driver and module load failures.

Commands:

```bash
systemctl --failed
journalctl -b -p warning
journalctl -u cloud-init -u cloud-final
journalctl -k
lsmod
uname -a
```

## Layer 3: Runtime and kubelet

```bash
systemctl status containerd kubelet
journalctl -u containerd -u kubelet --since -30m
crictl info
crictl ps -a
```

Check:

- cgroup driver mismatch.
- Runtime socket.
- Sandbox-image availability.
- Certificate bootstrap.
- Authentication and authorization.
- Kubelet configuration.

## Layer 4: Kubernetes

```bash
kubectl describe node <node>
kubectl get events --all-namespaces --sort-by=.lastTimestamp
kubectl get pods -A -o wide --field-selector spec.nodeName=<node>
```

Check node conditions, taints, allocatable resources, DaemonSets, and pod sandbox errors.

## Layer 5: Networking and storage

- CNI logs.
- IP allocation.
- Routes.
- MTU.
- conntrack.
- CSI logs.
- Volume attachment events.

## Layer 6: Compare candidate and control

Run the same diagnostic bundle on a known-good node and diff:

- Packages.
- Kernel parameters.
- systemd unit dependencies.
- container runtime config.
- kubelet config.
- loaded modules.
- routes and interfaces.
- resource baseline.

This comparative method is often faster than investigating the candidate in isolation.

---

# 21. Common failure modes

## Failure 1: Node joins but CNI never becomes ready

Possible causes:

- Missing kernel module.
- Incorrect sysctl.
- Cgroup mismatch.
- CNI binary or mount path changed.
- Security hardening blocks BPF or netfilter access.
- systemd ordering race.

Mitigation:

- Keep node tainted and unschedulable.
- Stop rollout.
- Replace with known-good image.

## Failure 2: Node is Ready but pods cannot pull images

Possible causes:

- Missing CA certificate.
- Broken DNS.
- ECR credential-provider regression.
- Proxy configuration.
- MTU issue.
- Node-role permission change.

## Failure 3: Random pod failures only at high density

Possible causes:

- conntrack limit.
- PID limit.
- file-descriptor limit.
- pod-IP exhaustion.
- memory baseline increased.
- ephemeral-storage pressure.

## Failure 4: Drain hangs

Possible causes:

- PDBs.
- Termination hooks.
- finalizers.
- local storage.
- DaemonSet handling.
- kubelet or runtime shutdown bug.

## Failure 5: Works on one instance family but not another

Possible causes:

- Driver differences.
- ENA/NVMe behavior.
- architecture mismatch.
- bootstrap max-pod calculation.
- GPU or accelerator stack.

## Failure 6: Autoscaling amplifies the defect

A broken image may launch repeatedly when capacity is already constrained.

Controls:

- Launch-failure circuit breaker.
- Maximum candidate capacity.
- Automatic fallback to known-good node class.
- Per-AZ and per-cell rollout limits.
- Alerts on repeated registration failure.

---

# 22. Immediate mitigation versus permanent correction

## Immediate mitigation

- Halt promotion.
- Disable candidate launch-template or node-class version.
- Restore known-good provisioning.
- Add replacement capacity.
- Cordon and drain candidate nodes.
- Protect critical workloads from further movement.
- Preserve failed nodes or logs for evidence.

## Permanent correction

- Fix the image build or configuration source.
- Add a regression test reproducing the failure.
- Expand the compatibility matrix if the failure was hardware-specific.
- Improve stop conditions.
- Improve rollout blast-radius limits.
- Update the rollback rehearsal.
- Record the learning in the image qualification standard.

A mature organization does not close the incident after replacing nodes; it converts the failure into a permanent test.

---

# 23. Trade-offs

## Bake more versus configure at boot

Bake more:

- Faster and more deterministic scale-out.
- Better qualification fidelity.
- More image variants.

Configure at boot:

- Fewer images.
- More runtime dependency and drift.
- Slower and less predictable provisioning.

## Broad test matrix versus pipeline speed

A complete matrix is expensive.

Use risk-based tiers:

- Every build: static, boot, join, core networking.
- Every candidate: full platform integration.
- Kernel/runtime change: performance and resilience suite.
- Major OS change: broad compatibility and soak testing.

## Automatic versus manual promotion

Automation should enforce objective gates. Human approval remains useful for security exceptions, major changes, and business timing.

---

# 24. Observability requirements

Create dashboards by AMI ID and node-image version.

Node metrics:

- Launch count.
- Join success rate.
- Ready latency.
- NotReady transitions.
- Kubelet and runtime restarts.
- CPU, memory, disk, network baseline.
- OOM and eviction counts.
- Kernel warnings.
- CNI and CSI errors.
- Drain and termination time.

Workload metrics:

- Scheduling latency.
- Pod-start latency.
- Crash loops.
- Image-pull failures.
- DNS failures.
- Request latency and errors.
- Connection failures.

Always include image version as a low-cardinality dimension in node and rollout telemetry.

---

# 25. Security and governance

Required controls:

- Dedicated image-builder role.
- Least-privilege build account.
- Restricted AMI sharing.
- KMS encryption.
- Signed pipeline artifacts.
- Protected source repository and branch.
- Two-person approval for production promotion where required.
- Immutable audit trail.
- Expiration policy for old candidates.
- Approved rollback set.
- Regular revocation of vulnerable images.

Prevent workloads from selecting arbitrary AMIs through uncontrolled launch templates or node classes.

---

# 26. Common wrong answers

## Wrong answer: “The instance launched successfully, so the image is good.”

Boot success proves almost nothing about Kubernetes workload behavior.

## Wrong answer: “Run CIS scanning and promote if there are no critical CVEs.”

Security scanning does not validate networking, storage, kubelet behavior, performance, or drain safety.

## Wrong answer: “Deploy to staging, then production.”

This lacks measurable gates, representative workloads, canary limits, and rollback mechanics.

## Wrong answer: “Use rolling update and Kubernetes handles it.”

Kubernetes cannot guarantee replacement capacity, application disruption safety, or that the new node image works.

## Wrong answer: “Patch the nodes if something breaks.”

This creates drift and destroys artifact-level confidence.

## Wrong answer: “Test only the largest production instance type.”

Driver, architecture, storage, and networking differences often make failures instance-family-specific.

---

# 27. Deep-dive interview answer

> I separate image qualification into artifact integrity, node correctness, Kubernetes correctness, workload correctness, and production safety. Artifact integrity proves exactly what was built and from which inputs. Node correctness proves boot, systemd, kernel, runtime, storage, network, reboot, and access behavior. Kubernetes correctness proves registration, certificate lifecycle, CNI, CSI, DNS, identity, capacity reporting, eviction, and drain. Workload correctness proves representative services can start, communicate, persist data, scale, terminate, and recover. Production safety proves the rollout is bounded, observable, reversible, and governed by customer SLIs.
>
> I promote the same immutable AMI through environments, not rebuild it. Production begins with a dedicated canary node group and selected workloads. The rollout controller compares candidate and control populations and halts on infrastructure or customer regression. I keep the previous image continuously compatible and predefine a replacement-based rollback. Every failure found in canary or production becomes a permanent qualification test.

---

# 28. Whiteboard architecture

```text
Git repository
   |
   v
Packer / Image Builder
   |
   +--> pinned base AMI
   +--> pinned packages
   +--> bootstrap source
   +--> hardening rules
   |
   v
Quarantine AMI + manifest + SBOM
   |
   +--> static inspection
   +--> vulnerability scan
   +--> boot matrix
   +--> EKS join test
   +--> CNI/CSI/DNS tests
   +--> workload lifecycle tests
   +--> performance comparison
   +--> fault injection
   |
   v
Promotion registry
   |
   +--> dev
   +--> staging
   +--> prod canary
   +--> progressive production
   |
   v
Known-good catalog + rollback pointer
```

At every stage show:

- Inputs.
- Test evidence.
- Promotion state.
- Stop conditions.
- Maximum blast radius.
- Rollback target.

---

# 29. Follow-up questions and model responses

## “Would you run every test for every small package update?”

> I would use risk-based test tiers, but every artifact still receives core static, boot, join, network, and workload-smoke validation. Kernel, runtime, CNI, storage, security, and bootstrap changes trigger the broad suite automatically.

## “How do you test Spot nodes?”

> I validate the same image on Spot-compatible instance families and explicitly test interruption handling, replacement capacity, drain timing, and autoscaler fallback. Spot behavior is part of rollout safety, not a separate afterthought.

## “What if the old AMI has a critical vulnerability?”

> I keep rollback compatibility but may prohibit rollback for workloads exposed to that vulnerability. The incident plan distinguishes availability rollback, security containment, and forward-fix. Sometimes the safest action is to halt expansion and accelerate a corrected image rather than restore the vulnerable image fleet-wide.

## “How long should canary soak?”

> Long enough to observe meaningful traffic, scheduled jobs, certificate behavior, scaling events, and at least one node lifecycle event. The gate should use both elapsed time and event volume. A quiet two-hour canary may provide less evidence than fifteen minutes under representative load.

## “How do you prevent a bad image from consuming all capacity?”

> Candidate node classes have strict maximum capacity, provisioning failure circuit breakers, and fallback to known-good capacity. Rollout proceeds one cell, AZ, or bounded node group at a time.

---

# 30. Hands-on lab

## Goal

Build and qualify a custom EKS worker image in a nonproduction AWS account.

## Lab stages

1. Pin an approved source AMI.
2. Build a versioned image with Packer.
3. Generate a manifest and package inventory.
4. Scan the image.
5. Launch an ephemeral instance and run static and boot checks.
6. Create an ephemeral EKS node group from the image.
7. Run networking, DNS, storage, identity, and workload tests.
8. Compare performance with the current image.
9. Create a tainted canary node group.
10. Move selected workloads to canary nodes.
11. Inject a controlled failure.
12. Execute replacement-based rollback.
13. Record test evidence and promotion status.

## Required evidence

- Packer manifest.
- AMI ID.
- SBOM.
- Package diff.
- Boot timing.
- Node-ready timing.
- Kubernetes test results.
- Performance comparison.
- Canary dashboard.
- Rollback duration.
- Post-test findings.

---

# 31. Mapping to Nathanel's experience

Use your Azure and Kubernetes platform work as the bridge:

> In my Azure candidate platform I treated AKS infrastructure, node identity, networking, ingress, CSI secrets, and RBAC as one integrated system rather than independent Terraform resources. I would apply the same production discipline to EKS node images: qualify the complete node lifecycle and every platform dependency, not merely prove that the VM boots. My background troubleshooting Linux, Kubernetes networking, storage, system services, and large-scale automation gives me the mechanical depth to design both the image pipeline and the failure investigation process.

You can also map your large MySQL import work to artifact discipline:

> The 45-terabyte import reinforced that deterministic inputs, resumability, validation checkpoints, and controlled failure recovery matter more than optimistic automation. An AMI promotion pipeline uses the same principles: immutable inputs, explicit checkpoints, measurable correctness, and a known recovery point.

---

# 32. Memorization card

```text
CUSTOM AMI = PLATFORM RELEASE

1. Pin source and packages.
2. Build immutable artifact.
3. Record manifest, SBOM, digest, versions.
4. Static inspect.
5. Boot across architecture and instance matrix.
6. Join ephemeral cluster.
7. Test CNI, CSI, DNS, identity, observability.
8. Test realistic workloads and drain/reboot.
9. Compare performance to known-good image.
10. Inject failures.
11. Promote same artifact through environments.
12. Canary with taints and bounded capacity.
13. Gate on customer and platform SLIs.
14. Roll back by replacement, not drift-inducing patching.
15. Convert every failure into a permanent test.
```

---

# 33. Final Principal-level principle

> The purpose of image qualification is not to prove that the image works once. It is to prove that the platform can safely create, operate, observe, replace, and recover nodes built from that image without allowing a single artifact defect to become a fleet-wide outage.
