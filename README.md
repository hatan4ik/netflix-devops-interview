# Netflix-Scale DevOps Interview Curriculum

A public, principal-level training repository for advanced DevOps, SRE, Platform Engineering, Kubernetes, cloud, Linux, incident response, chaos engineering, and technical-leadership interviews.

> These are hypothetical interview scenarios and engineering exercises. They are not a claim about Netflix's private production architecture or its actual interview process.

> **Canonical shared foundations:** Reusable Linux, Kubernetes, networking, service-mesh, eBPF, cloud, Terraform, identity, autoscaling, observability, reliability, and leadership chapters are consolidated in [`hatan4ik/staff-sre-platform-engineering-handbook`](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook). This repository remains the Netflix/media-delivery interview track. Existing foundational chapters here are migration sources until equivalent canonical coverage is verified.

## Repository contents

### Comprehensive training pack

- [`docs/TRAINING_PACK.md`](docs/TRAINING_PACK.md) — full 17-scenario interview training pack.
- [`docs/FAANG_BOARD_REVIEW.md`](docs/FAANG_BOARD_REVIEW.md) — Staff/Principal engineering-board review and corrections.
- [`docs/FAANG_BAR_RAISER_ARCHITECTURE_ADDENDUM.md`](docs/FAANG_BAR_RAISER_ARCHITECTURE_ADDENDUM.md) — hyperscale failure-mode and mechanical-sympathy addendum.

### Ground-up curriculum and migration sources

- [`curriculum/01-fine-grained-service-discovery.md`](curriculum/01-fine-grained-service-discovery.md) — Question 1 Netflix/media adapter and original migration source for large-scale Envoy/Istio discovery.
- [`curriculum/02-ebpf-cilium-runtime-security.md`](curriculum/02-ebpf-cilium-runtime-security.md) — Question 2: original deep source material for eBPF/Cilium runtime security.
- [`curriculum/03-multicloud-routing-identity-secrets.md`](curriculum/03-multicloud-routing-identity-secrets.md) — Question 3 multi-cloud scenario; reusable workload-identity theory is now canonical in the shared handbook, while cross-cloud routing and regional secret-serving details remain migration sources.
- [`curriculum/04-eks-systemd-node-failure-repair.md`](curriculum/04-eks-systemd-node-failure-repair.md) — Question 4: EKS node failure detection, fencing, repair, and replacement.
- [`curriculum/05-custom-ami-validation-promotion.md`](curriculum/05-custom-ami-validation-promotion.md) — Question 5: immutable custom AMI construction, qualification, canary promotion, stop conditions, and replacement-based rollback.
- [`curriculum/06-business-aware-kubernetes-probes.md`](curriculum/06-business-aware-kubernetes-probes.md) — Question 6: startup, liveness, readiness, capability health, graceful drain, overload protection, and probe-safe rollout design.
- [`curriculum/07-dns-outage-service-mesh.md`](curriculum/07-dns-outage-service-mesh.md) — Question 7: layered DNS failure isolation across application resolvers, NodeLocal DNSCache, CoreDNS, service routing, conntrack, network policy, and mesh DNS capture.
- [`curriculum/08-terraform-backend-state-lock-failure.md`](curriculum/08-terraform-backend-state-lock-failure.md) — Question 8: remote-state integrity, S3/DynamoDB locking, stale-lock investigation, safe force-unlock, reconciliation, recovery, and CI concurrency controls.
- [`curriculum/09-mtls-failure-after-envoy-rollout.md`](curriculum/09-mtls-failure-after-envoy-rollout.md) — Question 9: mTLS rollout failures across certificates, SDS, trust bundles, SPIFFE identities, PeerAuthentication, DestinationRule, AuthorizationPolicy, xDS, and mixed proxy revisions.
- [`curriculum/10-hpa-not-scaling-high-cpu.md`](curriculum/10-hpa-not-scaling-high-cpu.md) — Question 10 Netflix traffic-spike adapter; reusable autoscaling control-loop and capacity-realization theory is now canonical in the shared handbook.
- [`curriculum/11-cache-sidecar-tail-latency.md`](curriculum/11-cache-sidecar-tail-latency.md) — Question 11: per-Pod cache topology, tail-latency decomposition, cold-cache amplification, stampede control, resource contention, connection pools, retry budgets, and origin protection.
- [`curriculum/12-504s-while-health-checks-green.md`](curriculum/12-504s-while-health-checks-green.md) — Question 12: identifying the 504 issuer, tracing exhausted deadlines, separating health from business-path success, controlling retries and queues, streaming diagnostics, and graceful degradation.
- [`curriculum/13-sudden-nat-gateway-cost-increase.md`](curriculum/13-sudden-nat-gateway-cost-increase.md) — Question 13: NAT billing forensics, CloudWatch correlation, Flow Log top-talkers, workload attribution, endpoint regressions, cross-AZ routing, retry amplification, egress security, and durable cost controls.
- [`curriculum/14-slo-error-budget-ownership.md`](curriculum/14-slo-error-budget-ownership.md) — Question 14: user-journey SLIs, error-budget mathematics, multi-window burn alerts, denominator integrity, protected slices, SLO-as-code, ownership, policy, adoption, and cultural change.
- [`curriculum/15-multi-region-failover-without-dns-only.md`](curriculum/15-multi-region-failover-without-dns-only.md) — Question 15: stable global traffic steering, RTO/RPO, active-passive design, write fencing, replication eligibility, dependency and capacity readiness, long-lived connections, failover, and controlled failback.

## Shared canonical prerequisites

- [Linux Internals module](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/tree/main/core/linux)
  - [Architecture, boot, PID 1, systemd, and syscalls](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/core/linux/01-architecture-boot-syscalls.md)
  - [Processes, scheduling, cgroup CPU, interrupts, and load](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/core/linux/02-processes-scheduler.md)
  - [Memory, page cache, NUMA, reclaim, PSI, and OOM](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/core/linux/03-memory.md)
  - [VFS, filesystems, block I/O, NVMe, and latency](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/core/linux/04-storage-io.md)
  - [Networking, namespaces, cgroups, containers, and Linux security](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/core/linux/05-networking-containers-security.md)
  - [Observability, profiling, eBPF, and production debugging](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/core/linux/06-observability-debugging.md)
  - [Linux incident labs and production scenarios](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/core/linux/07-linux-incident-labs.md)
- [Canonical workload identity, federation, and SPIFFE chapter](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/core/security/identity/workload-identity-federation.md)
- [Canonical Kubernetes autoscaling and capacity-realization chapter](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/core/kubernetes/autoscaling/control-loops-capacity-realization.md)
- [Canonical service-mesh, Envoy, Istio, and xDS module](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/tree/main/core/service-mesh)
  - [Fine-grained service discovery across large service estates](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/core/service-mesh/fine-grained-service-discovery.md)
- [Canonical eBPF, Cilium, Hubble, Falco, and Tetragon chapter](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/core/ebpf-security/cilium-hubble-falco-tetragon.md)
- [Canonical Terraform state integrity chapter](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/core/infrastructure-as-code/terraform-state-integrity.md)
- [Canonical GitOps and progressive delivery chapter](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/core/delivery-gitops/gitops-progressive-delivery.md)
- [Consolidated curriculum map](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/curriculum-map.md)
- [Migration and ownership plan](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/MIGRATION_PLAN.md)

New reusable foundations must be written in the shared handbook. New files here should focus on Netflix/media-specific questions, answer adapters, playback failure modes, streaming traffic patterns, and mock interviews.

## The 17-scenario sequence

1. Fine-grained discovery across 1,000+ services.
2. eBPF and Cilium runtime network security.
3. Multi-cloud routing, identity, and secrets.
4. Intermittent systemd failures on EKS nodes.
5. Custom AMI validation and promotion.
6. Business-aware Kubernetes probes.
7. DNS outage in a service mesh.
8. Terraform backend and state-lock failure.
9. mTLS failure after an Envoy or mesh rollout.
10. HPA not scaling under high reported CPU.
11. Cache sidecar increasing tail latency.
12. Requests returning 504 while health checks remain green.
13. Sudden NAT Gateway cost increase.
14. Establishing SLO and error-budget ownership.
15. Multi-region failover without relying only on DNS.
16. Chaos engineering and graceful degradation for a major release.
17. Demonstrating modernization ROI.

## Core principle

> At hyperscale, the objective is not to prevent every component failure. It is to ensure that no component, retry policy, health check, repair controller, state lock, identity bridge, or autoscaling loop can expand a local fault beyond its intended failure domain.
