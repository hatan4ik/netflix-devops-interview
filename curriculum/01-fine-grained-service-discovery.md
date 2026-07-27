# Netflix Adapter — Fine-Grained Service Discovery at Streaming Scale

> **Original scenario:** A platform has thousands of microservices across many Kubernetes clusters and regions. Design service discovery that avoids full-mesh configuration fan-out, preserves last-known-good routing during control-plane failure, and supports safe regional and service-level isolation.

This file is intentionally a **thin Netflix/media-delivery adapter**. Reusable Envoy, Istio, xDS, service-registry, convergence, control-plane, data-plane, and multi-cluster theory is canonical in the shared handbook.

## Canonical prerequisites

Read these first:

- [Fine-grained service discovery with Envoy, Istio, and xDS](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/core/service-mesh/fine-grained-service-discovery.md)
- [Envoy request-path, timeout, reset, and 504 debugging](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/core/service-mesh/envoy-request-path-debugging.md)
- [Service-mesh mTLS, SDS, DNS capture, and multi-cluster reliability](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/core/service-mesh/mtls-sds-dns-multicluster.md)
- [Kubernetes Service, DNS, Ingress, and Gateway request paths](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/core/kubernetes/networking/service-dns-ingress-gateway-request-path.md)
- [Graceful degradation, overload control, and blast radius](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/core/reliability/graceful-degradation-overload-blast-radius.md)

## What the interviewer is testing

The Netflix-shaped part is whether you can apply those foundations to a very large media-delivery estate:

- thousands of independently deployed services;
- many clusters, regions, cells, and network boundaries;
- playback-critical and noncritical dependency classes;
- high release frequency and configuration churn;
- tail-latency sensitivity;
- control-plane failure that must not interrupt established playback paths;
- explicit regional evacuation and graceful-degradation behavior.

## Ninety-second answer

> I would not distribute every service and endpoint to every proxy. I would make discovery dependency-aware: each workload receives only the clusters, routes, identities, and endpoints it is authorized and expected to call. Service metadata defines ownership, criticality, locality, failover eligibility, and data sensitivity. That reduces xDS fan-out, proxy memory, configuration blast radius, and accidental coupling.
>
> I would separate the authoritative registry from regional discovery control planes and data-plane proxies. Regional control planes consume validated registry state, compile scoped xDS resources, and publish versioned snapshots. Proxies ACK valid state, NACK invalid updates, retain last-known-good configuration, and reconnect with jitter. The system measures registry-to-proxy convergence and stale-state age rather than calling control-plane uptime alone success.
>
> Playback-critical traffic prefers same-cell or same-region dependencies. Cross-region fallback is explicit and allowed only when destination capacity, data authority, identity, and latency support it. Optional personalization, recommendation, artwork enrichment, and analytics degrade before manifest, entitlement, DRM, and segment-delivery paths. A bad service, cluster, or discovery release is isolated by cell and rollout ring.
>
> I would validate it through xDS rejection, endpoint churn, control-plane outage, regional isolation, DNS failure, gateway loss, and failover game days. Recovery is proven by playback-start, rebuffer, and request-path SLIs plus fresh accepted configuration at the affected proxies.

## Netflix-specific architecture

```text
service catalog and ownership metadata
        |
        v
validated global registry events
        |
        +-------------------------------+
        |                               |
        v                               v
regional discovery control plane A     regional discovery control plane B
        | dependency-scoped xDS                | dependency-scoped xDS
        v                               v
cell / cluster proxies A                cell / cluster proxies B
        |                               |
        v                               v
playback, entitlement, DRM, manifest, metadata, and optional services
```

### Dependency classes

| Class | Examples | Failure behavior |
|---|---|---|
| Playback safety and authorization | entitlement, DRM policy, account restrictions | fail according to security and rights invariants; never silently bypass |
| Playback establishment | manifest, session, device capability | protected capacity, same-region preference, bounded failover |
| Media delivery | origin selection, segment and byte-range paths | CDN/origin fallback, locality, capacity and cache awareness |
| Experience enrichment | recommendations, artwork variants, social context | omit, cache, or serve stale data |
| Analytics and background work | event enrichment, batch, experimentation | queue, sample, shed, or replay later |

## Discovery contract

Every exported service should declare:

- owner and escalation path;
- stable service identity;
- allowed callers;
- ports and protocols;
- locality and topology;
- criticality class;
- cross-cell and cross-region eligibility;
- timeout and retry owner;
- degraded-mode behavior;
- data and write-authority constraints;
- SLO and synthetic journey;
- rollback and retirement policy.

## Scale strategy

Do not answer with an arbitrary proxy count. Explain the dimensions:

```text
configuration load approximately grows with
  proxies receiving a resource
  × resource count
  × update frequency
  × serialization and validation cost
```

Bound it through dependency-scoped configuration, hierarchical distribution, incremental xDS where appropriate, versioned snapshots, deduplication, locality, service export policy, update batching, slow-consumer detection, and canary control-plane/proxy revisions.

## Failure handling

### Registry or control-plane loss

- proxies continue with last-known-good state;
- configuration age and certificate expiry are visible;
- no global restart or synchronized reconnect;
- critical updates are prioritized after recovery;
- established playback paths continue for the tested isolation window.

### Invalid configuration

- affected proxies NACK it;
- the prior accepted version remains active;
- rollout stops by revision and cell;
- validation reproduces the rejection before retrying;
- no operator deletes working state to force convergence.

### Regional or cell failure

- route only eligible services and cohorts;
- verify remote capacity and data/write authority;
- prevent retries from multiplying shifted traffic;
- degrade optional calls first;
- fail back gradually after local state is healthy and backlog is controlled.

## Evidence and SLOs

Track registry-event-to-proxy convergence, xDS ACK/NACK, active version, stale configuration and endpoint age, proxy memory and resource count, control-plane queue depth, slow consumers, endpoint locality, no-healthy-upstream and local-reply rate, cross-region traffic, playback-start success/latency, rebuffer rate, and fatal playback-error cohorts.

## Whiteboard flow

```text
1. classify playback-critical versus optional dependencies
2. declare service export, identity, locality, and failure contract
3. distribute only authorized dependency state
4. validate and version every snapshot
5. retain last-known-good state on NACK or control-plane loss
6. isolate by cell, region, service, and rollout ring
7. fail over only with capacity, identity, and data-authority checks
8. prove recovery through playback and proxy-convergence SLIs
```

## Adversarial follow-ups

### Why not give every proxy the complete catalog?

It increases memory, update fan-out, convergence time, security exposure, and the blast radius of unrelated service changes.

### What happens when the control plane is down for hours?

Existing accepted configuration continues to serve. The design must monitor stale endpoint state, certificate expiry, and changed safety policy; last-known-good is bounded, not permanent independence.

### How do you prevent a global reconnect storm?

Exponential backoff with jitter, hierarchical distribution, bounded concurrent streams, slow-consumer controls, and prioritized recovery by criticality and cell.

### Should every playback dependency fail over cross-region?

No. Remote use is allowed only when latency, capacity, data semantics, identity, and write authority are safe. Some dependencies should degrade or fail closed instead.

## Personal-story bridge

Use a truthful example involving a large heterogeneous estate, standardizing ownership or configuration, reducing operational fan-out, isolating a regional/network failure, preserving service during control-plane degradation, or creating a reusable rollout/diagnostic mechanism. State production facts separately from this hypothetical streaming-scale architecture.

## Practical drill

Run the shared [service-mesh reliability contract lab](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/tree/main/labs/service-mesh/01-xds-mtls-dns-failover) and explain how its ACK/NACK, stale-DNS, trust-rotation, failover, and retry invariants map to a playback-critical service graph.
