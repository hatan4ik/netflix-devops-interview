# Netflix Track — Canonical Ownership and Adapter Map

This repository owns Netflix/media-delivery interview context. Reusable engineering explanations belong in [`hatan4ik/staff-sre-platform-engineering-handbook`](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook).

## Decision rule

A curriculum file remains deep only when the material is materially specific to streaming, playback, CDN/origin behavior, media security, experimentation, or a Netflix-shaped organizational decision. Generic Linux, Kubernetes, Terraform, service-mesh, reliability, observability, security, and distributed-systems theory should be linked rather than forked.

## Parity review

| Netflix curriculum chapter | Canonical owner | Adapter decision |
|---|---|---|
| `01-fine-grained-service-discovery.md` | `core/service-mesh/` plus Kubernetes networking and reliability | **Thinned.** Retains playback-critical dependency classes, discovery scale, cross-region eligibility, and Netflix-shaped SLOs. |
| `02-ebpf-cilium-runtime-security.md` | `core/ebpf-security/`, Linux, Kubernetes networking, and node images | **Thinned.** Retains streaming dataplane risk, playback capacity, migration rings, and security evidence priorities. |
| `03-multicloud-routing-identity-secrets.md` | identity, secrets, service mesh, networking, DR | **Retain deep temporarily.** It still combines cloud-routing and multi-cloud operational choices not yet represented by one canonical cloud-routing chapter. |
| `04-eks-systemd-node-failure-repair.md` | Kubernetes node lifecycle, runtime, node images, probes, storage, and Linux | **Thinned.** Retains playback-capacity guards, node-repair circuit breakers, and media-path recovery proof. |
| `05-secret-management.md` | `core/security/secrets/` and workload identity | **Retain as Netflix adapter pending final parity pass.** Preserve media-service secret classes, operational migration, and high-scale rotation context. |
| `06-custom-admission-controllers.md` | platform policy, Kubernetes admission, supply-chain security | **Retain as Netflix adapter pending final parity pass.** Preserve organization-specific policy ownership and rollout framing. |
| `07-business-aware-probes.md` | Kubernetes workload lifecycle and reliability overload/degradation | **Retain Netflix-specific sections.** Playback/business readiness and degradation semantics remain track value. Generic probe theory should link to core. |
| `08-debugging-kubernetes-dns.md` | Kubernetes networking and service-mesh DNS | **Retain Netflix incident adapter.** Preserve streaming request-path, regional, client, and DNS-cache cohorts. |
| `09-terraform-state-drift-recovery.md` | Terraform state integrity and IaC governance | **Retain only AWS/account/Netflix operational examples; generic recovery is canonical.** Candidate for a later mechanical thin pass. |
| `10-kubernetes-hpa-scaling.md` | Kubernetes autoscaling and overload/capacity | **Retain traffic-shape and playback-demand adapter; generic control-loop theory is canonical.** |
| `11-os-kernel-security-patching.md` | Linux, node images, supply chain, node lifecycle | **Retain operational patch strategy and Netflix release context; link shared kernel/image theory.** |
| `12-slo-error-budget.md` | reliability SLO/error-budget module | **Retain playback SLI and product-policy adapter; generic mathematics and governance are canonical.** |
| `13-cache-sidecar-tail-latency.md` | distributed systems caching, overload, observability, service mesh | **Track-owned.** Cache-sidecar and media tail-latency scenario is intentionally Netflix-shaped. |
| `14-debugging-504-partial-failures.md` | incident request paths/cohorts, observability, Envoy debugging | **Track-owned adapter.** Playback 504, manifest, entitlement, DRM, CDN, and partial-user reasoning remain Netflix-specific. |
| `15-modernization-roi.md` | future leadership/economics core plus platform product | **Track-owned.** Modernization investment, sequencing, migration economics, and executive framing remain interview context. |

## No-duplication standard

Every future curriculum change must:

1. search the canonical handbook first;
2. link the exact shared chapter;
3. keep only the Netflix/media-delivery question, assumptions, answer adapter, trade-offs, drills, and story bridge;
4. avoid copying generic command catalogs or textbook explanations;
5. preserve truthful production evidence separately from hypothetical streaming-scale architecture.

## Canonical entry points

- [Kubernetes](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/tree/main/core/kubernetes)
- [Service mesh](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/tree/main/core/service-mesh)
- [Security](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/tree/main/core/security)
- [Observability](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/tree/main/core/observability)
- [Reliability](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/tree/main/core/reliability)
- [Incident response](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/tree/main/core/incident-response)
- [Distributed systems](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/tree/main/core/distributed-systems)
- [Platform engineering](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/tree/main/core/platform-engineering)
- [Executable labs](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/tree/main/labs)
