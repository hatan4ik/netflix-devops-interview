# Netflix-Scale DevOps Interview Curriculum

A public, principal-level training repository for advanced DevOps, SRE, Platform Engineering, Kubernetes, cloud, Linux, incident response, chaos engineering, and technical-leadership interviews.

> These are hypothetical interview scenarios and engineering exercises. They are not a claim about Netflix's private production architecture or its actual interview process.

> **Canonical shared foundations:** Reusable Linux, Kubernetes, networking, service-mesh, eBPF, cloud, Terraform, observability, reliability, and leadership chapters are being consolidated in [`hatan4ik/staff-sre-platform-engineering-handbook`](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook). This repository remains the Netflix/media-delivery interview track. Existing foundational chapters here are migration sources until equivalent canonical coverage is verified.

## Repository contents

### Comprehensive training pack

- [`docs/TRAINING_PACK.md`](docs/TRAINING_PACK.md) — full 17-scenario interview training pack.
- [`docs/FAANG_BOARD_REVIEW.md`](docs/FAANG_BOARD_REVIEW.md) — Staff/Principal engineering-board review and corrections.
- [`docs/FAANG_BAR_RAISER_ARCHITECTURE_ADDENDUM.md`](docs/FAANG_BAR_RAISER_ARCHITECTURE_ADDENDUM.md) — hyperscale failure-mode and mechanical-sympathy addendum.

### Ground-up curriculum and migration sources

- [`curriculum/01-fine-grained-service-discovery.md`](curriculum/01-fine-grained-service-discovery.md) — Question 1: fine-grained service discovery across 1,000+ services with Envoy/Istio.
- [`curriculum/02-ebpf-cilium-runtime-security.md`](curriculum/02-ebpf-cilium-runtime-security.md) — Question 2: Linux eBPF foundations, Cilium/Hubble networking, Falco detection, and Tetragon runtime enforcement.
- [`curriculum/03-multicloud-routing-identity-secrets.md`](curriculum/03-multicloud-routing-identity-secrets.md) — Question 3: multi-cloud routing, workload identity federation, and secret delivery.
- [`curriculum/04-eks-systemd-node-failure-repair.md`](curriculum/04-eks-systemd-node-failure-repair.md) — Question 4: EKS node failure detection, fencing, repair, and replacement.

New reusable foundations will be written in the shared handbook. New files here should focus on Netflix/media-specific questions, answer adapters, playback failure modes, and mock interviews.

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

> At hyperscale, the objective is not to prevent every component failure. It is to ensure that no component, retry policy, health check, repair controller, or state lock can expand a local fault beyond its intended failure domain.
