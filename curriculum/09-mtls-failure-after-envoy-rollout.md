# Chapter 9 — mTLS Failure After an Envoy or Service-Mesh Rollout

## Interview scenario

A routine Envoy or service-mesh rollout completes successfully according to the deployment controller, but traffic begins failing between selected services. Some requests return `503`, `UF`, `URX`, or TLS handshake errors. Health checks remain green, failures are asymmetric, and only particular namespaces, clusters, revisions, or workload versions are affected.

You are the Staff/Principal SRE leading diagnosis, containment, recovery, and prevention.

---

## 1. What the interviewer is testing

A strong answer must show that you can:

- separate application, transport, proxy, identity, policy, and control-plane failures;
- reason about both sides of an mTLS connection;
- distinguish certificate issuance from certificate delivery and certificate use;
- avoid broad security downgrades while restoring traffic;
- debug mixed-version and mixed-policy rollouts;
- define safe rollout gates, rollback conditions, and durable controls;
- communicate clearly during an incident where the visible symptom is far from the root cause.

The weak answer is: “Restart Istio” or “set mTLS to permissive.”

The Principal answer is: “First identify which identity, trust, policy, or data-plane transition failed; freeze expansion; preserve evidence; restore the smallest affected failure domain; then prove both transport security and application correctness before continuing.”

---

## 2. Build the end-to-end mental model

For service A to call service B through a mesh, several independent systems must agree:

1. **Workload identity**
   - The caller must possess a valid identity certificate and private key.
   - The receiver must possess a valid server identity certificate and private key.
   - Identities usually encode SPIFFE-compatible URIs such as:

```text
spiffe://cluster.local/ns/payments/sa/checkout
```

2. **Trust material**
   - Both proxies must trust the appropriate root or intermediate CA.
   - Multi-cluster or multi-trust-domain deployments may require trust-domain aliases or federated bundles.

3. **Certificate delivery**
   - Envoy typically receives secrets dynamically through SDS.
   - A certificate may exist in the CA but never arrive at the sidecar.

4. **Traffic policy**
   - `PeerAuthentication` controls inbound mTLS expectations.
   - `DestinationRule` controls outbound TLS behavior.
   - `AuthorizationPolicy` controls whether the authenticated principal may call the destination.

5. **Listener and cluster configuration**
   - LDS must create the right inbound and outbound listeners.
   - CDS/EDS must deliver the correct upstream cluster and endpoints.
   - Route configuration must select the expected cluster.

6. **Network path**
   - TCP must reach the destination proxy.
   - NetworkPolicy, security groups, CNI, conntrack, NAT, and load balancers must not interrupt the handshake.

7. **Time**
   - Certificate validity depends on synchronized clocks.
   - `NotBefore`, `NotAfter`, and rotation windows matter.

mTLS therefore fails when any one of these planes disagrees with the others.

---

## 3. Recognize the major failure classes

### 3.1 Caller does not originate TLS

The destination requires mTLS, but the caller sends plaintext.

Typical causes:

- `PeerAuthentication` changed to `STRICT` before callers were updated;
- a `DestinationRule` forces `DISABLE`;
- sidecar injection is missing on the caller;
- traffic bypasses Envoy through an excluded port or direct pod IP path;
- a mesh revision changed auto-mTLS behavior.

Expected symptoms:

- destination proxy rejects the connection;
- handshake errors appear only for specific callers;
- application containers may never see the request.

### 3.2 Caller uses TLS but destination expects plaintext

The caller originates mTLS while the destination port is not intercepted or is intentionally plaintext.

Typical causes:

- incorrect `DestinationRule` with `ISTIO_MUTUAL`;
- port naming or protocol detection changes;
- service port maps to the wrong target port;
- destination workload is outside the mesh;
- a gateway or legacy backend does not support mesh mTLS.

### 3.3 Certificate is absent, expired, or not yet valid

Typical causes:

- SDS stream failure;
- CA or istiod outage;
- stale sidecar after rotation;
- clock skew;
- short TTL plus delayed refresh;
- file descriptor, memory, or thread exhaustion in Envoy;
- certificate request rejected because of workload identity or token issues.

### 3.4 Trust bundle mismatch

Typical causes:

- root CA rollover performed non-overlappingly;
- one cluster uses an old root;
- intermediate chain is incomplete;
- trust-domain migration lacks aliases;
- external CA integration delivers a different chain than expected;
- control planes from different revisions publish incompatible bundles.

### 3.5 Identity or SAN mismatch

The peer presents a valid certificate, but it represents the wrong identity.

Typical causes:

- service account changed during rollout;
- namespace or service-account selector mismatch;
- `subjectAltNames` in `DestinationRule` is too restrictive or incorrect;
- traffic unexpectedly resolves to a workload in another cluster or namespace;
- ambient/sidecar migration changes the identity hop.

### 3.6 Authorization failure mistaken for TLS failure

The TLS handshake succeeds, but `AuthorizationPolicy` denies the authenticated principal.

Symptoms may still look like connectivity failure to the application. Distinguish:

- handshake failure;
- RBAC denial;
- upstream reset;
- route or cluster absence.

### 3.7 Mixed data-plane or control-plane revisions

During a canary mesh upgrade, old and new proxies may differ in:

- TLS defaults;
- cipher suites;
- xDS resource interpretation;
- SDS behavior;
- protocol sniffing;
- auto-mTLS logic;
- trust-bundle format;
- retry and connection-pool behavior.

Failures are often directional: new caller to old server works, but old caller to new server fails.

---

## 4. First-response incident strategy

### Step 1 — Stop expansion

Immediately pause:

- mesh revision rollout;
- sidecar reinjection;
- control-plane upgrade;
- policy migration;
- certificate or CA rotation;
- application deployments that replace known-good proxies.

Do not let automation erase the comparison between healthy and unhealthy cohorts.

### Step 2 — Define the failure matrix

Build a matrix by:

- caller service;
- destination service;
- source namespace;
- destination namespace;
- source proxy version/revision;
- destination proxy version/revision;
- cluster and region;
- node pool;
- service account;
- protocol and port;
- new versus existing connections.

Example:

| Caller | Caller proxy | Destination | Destination proxy | Result |
|---|---|---|---|---|
| checkout-v1 | old | payments-v1 | old | success |
| checkout-v2 | new | payments-v1 | old | success |
| checkout-v1 | old | payments-v2 | new | fail |
| checkout-v2 | new | payments-v2 | new | fail |

This suggests the destination-side rollout is more likely than the caller-side rollout.

### Step 3 — Separate existing from new connections

mTLS configuration changes often affect only new connections. Existing HTTP/2 or keepalive connections may remain healthy.

Ask:

- Do failures appear only after connection churn?
- Does restarting a caller make the problem worse?
- Are long-lived connections masking the outage?
- Did a load balancer, proxy rollout, or connection-age setting force synchronized reconnects?

Avoid mass restarts until you understand this distinction.

### Step 4 — Capture evidence before rollback

Collect from healthy and unhealthy proxies:

```bash
istioctl proxy-status
istioctl proxy-config secret <pod> -n <namespace>
istioctl proxy-config listeners <pod> -n <namespace>
istioctl proxy-config clusters <pod> -n <namespace>
istioctl proxy-config routes <pod> -n <namespace>
istioctl proxy-config endpoints <pod> -n <namespace>
istioctl analyze -A
```

Also collect:

```bash
kubectl logs <pod> -c istio-proxy --since=30m
kubectl get peerauthentication,destinationrule,authorizationpolicy -A -o yaml
kubectl get pods -A -o custom-columns='NS:.metadata.namespace,POD:.metadata.name,IMAGE:.spec.containers[*].image'
```

---

## 5. Envoy response flags and what they imply

Common flags include:

- `UF` — upstream connection failure;
- `URX` — upstream retry limit exceeded;
- `UH` — no healthy upstream;
- `NR` — no route configured;
- `UPE` — upstream protocol error;
- `DC` — downstream connection termination;
- `UC` — upstream connection termination;
- `LR` — local reset.

These flags narrow the layer, but do not prove the root cause. `UF` could be:

- TCP refusal;
- timeout;
- TLS handshake failure;
- certificate rejection;
- network policy;
- upstream listener mismatch.

Correlate access logs with Envoy debug logs, metrics, configuration dumps, and packet-level evidence.

---

## 6. Certificate and SDS debugging

### Inspect certificates loaded into Envoy

```bash
istioctl proxy-config secret checkout-abc123 -n shop
```

Validate:

- certificate exists;
- root bundle exists;
- serial number;
- issuer;
- SAN URI;
- expiration;
- whether healthy and unhealthy pods have different chains.

### Use the Envoy admin endpoint

```bash
kubectl exec -n shop checkout-abc123 -c istio-proxy -- \
  curl -s http://127.0.0.1:15000/certs
```

Useful endpoints:

```text
/certs
/config_dump
/clusters
/listeners
/stats
/server_info
```

### Check SDS and certificate metrics

Relevant metric families vary by version, but look for:

- SDS update failures;
- secret warming failures;
- expired certificate counts;
- days or seconds until expiration;
- xDS reject counters;
- control-plane disconnects;
- workload certificate request errors.

### Confirm clock synchronization

```bash
kubectl exec <pod> -c istio-proxy -- date -u
kubectl debug node/<node> -it --image=ubuntu -- chroot /host timedatectl status
```

A certificate can be valid centrally and rejected locally because one node is several minutes ahead or behind.

---

## 7. TLS handshake investigation

### Inspect from inside the workload network namespace

Use a debugging container or ephemeral pod with OpenSSL:

```bash
openssl s_client \
  -connect payments.shop.svc.cluster.local:8443 \
  -servername payments.shop.svc.cluster.local \
  -showcerts
```

For mesh mTLS, direct OpenSSL tests may not reproduce Envoy’s client-certificate behavior, but they can still prove:

- TCP reachability;
- server certificate chain;
- protocol and cipher negotiation;
- server-side plaintext/TLS mismatch.

### Packet capture

```bash
kubectl exec <pod> -c istio-proxy -- \
  tcpdump -i any -nn -s0 -w /tmp/mtls.pcap port 15006 or port 15001 or port 8443
```

A packet capture helps distinguish:

- SYN timeout;
- reset before handshake;
- TLS alert;
- successful handshake followed by application reset;
- MTU or fragmentation issue.

Interpret TLS alerts such as:

- unknown CA;
- bad certificate;
- certificate expired;
- handshake failure;
- protocol version;
- no shared cipher.

---

## 8. Istio policy interaction

### PeerAuthentication

Example strict namespace policy:

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: payments
spec:
  mtls:
    mode: STRICT
```

Before applying, prove every intended caller can originate mTLS and every destination port is correctly captured.

### DestinationRule

A problematic rule:

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: payments
  namespace: shop
spec:
  host: payments.payments.svc.cluster.local
  trafficPolicy:
    tls:
      mode: DISABLE
```

This can override auto-mTLS and break calls into a strict destination.

Explicit mesh TLS:

```yaml
trafficPolicy:
  tls:
    mode: ISTIO_MUTUAL
```

Explicit configuration is not always safer. It can become stale or conflict with service subsets and port-level policy. Prefer the simplest policy that correctly expresses intent.

### AuthorizationPolicy

Example source-principal restriction:

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: allow-checkout
  namespace: payments
spec:
  selector:
    matchLabels:
      app: payments
  rules:
    - from:
        - source:
            principals:
              - cluster.local/ns/shop/sa/checkout
```

If the rollout changes the service account or trust domain, TLS may succeed while authorization fails.

---

## 9. xDS and configuration-rejection failures

A proxy can remain running while rejecting a new configuration resource.

Check synchronization:

```bash
istioctl proxy-status
```

Look for stale or rejected:

- LDS;
- CDS;
- EDS;
- RDS;
- SDS.

Compare the intended and actual configuration:

```bash
istioctl proxy-config cluster <pod> -n <ns> --fqdn payments.payments.svc.cluster.local
istioctl proxy-config listener <pod> -n <ns>
istioctl proxy-config route <pod> -n <ns>
```

Control-plane success does not imply data-plane acceptance. A rollout gate should fail when canary proxies reject xDS resources even if pods are Ready.

---

## 10. Root and intermediate CA rotation

A safe CA rollover requires overlap.

### Unsafe sequence

1. Replace root A with root B.
2. Begin issuing certificates from B.
3. Some workloads still trust only A.
4. New-to-old or old-to-new handshakes fail.

### Safer sequence

1. Distribute a trust bundle containing A and B.
2. Verify every proxy has accepted the dual-root bundle.
3. Begin issuing leaf or intermediate certificates chaining to B.
4. Wait through certificate rotation and maximum connection lifetime.
5. Confirm no active identities depend on A.
6. Remove A in a separate controlled rollout.

For large fleets, define explicit gates:

- percentage of proxies with dual trust;
- certificate-chain distribution;
- handshake failure rate;
- oldest unrotated certificate;
- cross-cluster synthetic success;
- rollback window.

---

## 11. Mixed-version rollout strategy

A service-mesh upgrade should be treated as a distributed protocol migration.

### Canary dimensions

Canary by:

- control-plane revision;
- namespace;
- workload class;
- region;
- cluster;
- inbound-heavy versus outbound-heavy services;
- HTTP/1.1, HTTP/2, gRPC, and raw TCP;
- internal and external destinations.

### Compatibility matrix

Test all important pairs:

```text
old client -> old server
old client -> new server
new client -> old server
new client -> new server
```

Also test gateway paths and cross-cluster calls.

### Stop conditions

Stop automatically on:

- TLS handshake errors above baseline;
- xDS rejection;
- SDS delivery delay;
- certificate-expiration risk;
- `503`/`UF` increase;
- authorization denial increase;
- latency regression from connection churn;
- unexpected plaintext traffic;
- synthetic identity mismatch.

---

## 12. Safe mitigation hierarchy

Choose the smallest reversible action.

1. Pause rollout and preserve healthy cohorts.
2. Route traffic away from the broken destination revision.
3. Roll back the new sidecar or control-plane revision.
4. Correct the offending policy in the narrow namespace, workload, host, subset, or port.
5. Restore the previous trust bundle or dual-root overlap.
6. Repair SDS or control-plane connectivity.
7. Reissue certificates only for the affected cohort.
8. Use narrowly scoped `PERMISSIVE` mode only when required for recovery, time-bounded, monitored, and accompanied by an explicit path back to `STRICT`.

Avoid:

- global `PERMISSIVE`;
- disabling authorization globally;
- deleting all `DestinationRule` objects;
- restarting every proxy;
- rotating all certificates during diagnosis;
- forcing reconnection of healthy long-lived sessions.

A mitigation that restores availability by silently removing identity guarantees is not a complete recovery.

---

## 13. Observability and SLOs

Monitor at least:

### Transport security

- mTLS handshake success rate;
- plaintext versus mTLS connection ratio;
- TLS alert distribution;
- certificate expiration horizon;
- SDS update success and latency;
- xDS rejection counts;
- proxy-control-plane connectivity.

### Traffic outcome

- `503`, `UF`, `URX`, and reset rates;
- upstream connection establishment latency;
- connection reuse and churn;
- retry amplification;
- destination-specific success rate;
- cross-revision success matrix.

### Identity and policy

- authorization denials by principal;
- unexpected service-account identities;
- trust-domain mismatch;
- policy coverage gaps;
- workloads missing sidecars or using unexpected revisions.

### Suggested SLO

For critical service-to-service paths:

> 99.99% of connection attempts requiring mesh mTLS complete authentication and establish an upstream connection within the defined latency threshold, excluding explicitly approved maintenance windows.

Track the user-facing SLO separately. A secure handshake SLO does not replace an availability SLO.

---

## 14. Production-ready rollout controls

A mature platform should provide:

- revisioned control planes;
- namespace-level canaries;
- automated compatibility tests;
- policy validation in CI;
- admission checks for conflicting TLS modes;
- certificate-expiry alerts well before rotation deadlines;
- trust-bundle inventory;
- synthetic cross-service and cross-cluster mTLS tests;
- automated xDS/SDS rejection gates;
- rollback artifacts retained for every proxy version;
- documented emergency policy exceptions with owners and expiry times.

Policy-as-code should detect patterns such as:

- `STRICT` inbound policy plus outbound `DISABLE`;
- invalid SAN restrictions;
- global permissive mode;
- workloads selected by authorization policy but using unexpected service accounts;
- root removal before dual-trust convergence.

---

## 15. Debugging playbook

### Phase A — Scope

```bash
kubectl get pods -A -l security.istio.io/tlsMode=istio
istioctl proxy-status
kubectl get peerauthentication,destinationrule,authorizationpolicy -A
```

Determine whether the failure follows:

- source;
- destination;
- proxy revision;
- node;
- namespace;
- cluster;
- identity;
- port.

### Phase B — Configuration

```bash
istioctl proxy-config listeners <pod> -n <ns>
istioctl proxy-config clusters <pod> -n <ns>
istioctl proxy-config routes <pod> -n <ns>
istioctl proxy-config endpoints <pod> -n <ns>
```

### Phase C — Identity

```bash
istioctl proxy-config secret <pod> -n <ns>
kubectl exec <pod> -c istio-proxy -- curl -s localhost:15000/certs
```

Verify:

- identity;
- issuer;
- chain;
- root;
- validity window;
- healthy/unhealthy differences.

### Phase D — Runtime evidence

```bash
kubectl logs <pod> -c istio-proxy --since=20m | \
  grep -Ei 'tls|ssl|certificate|sds|rbac|handshake|upstream'
```

Inspect access-log flags and metrics.

### Phase E — Network

Test TCP, capture packets, and inspect network policy only after configuration and identity hypotheses are explicit.

### Phase F — Mitigate and verify

After a narrow rollback or policy repair, prove:

- new connections succeed;
- old connections remain stable;
- mTLS remains enabled;
- correct principal is observed;
- authorization succeeds;
- latency and retries return to baseline;
- all four old/new compatibility paths pass.

---

## 16. Hands-on lab

### Objective

Create and diagnose three distinct failures:

1. strict inbound mTLS with plaintext outbound policy;
2. authorization denial caused by service-account change;
3. trust-bundle or certificate-validity failure.

### Lab outline

1. Deploy client and server workloads with sidecars.
2. Confirm successful mTLS and inspect identities.
3. Apply namespace-level `STRICT` policy.
4. Add a conflicting `DestinationRule` with `DISABLE`.
5. Observe `503`/`UF`, then prove the destination never receives application traffic.
6. Repair the rule and verify new connections.
7. Change the client service account while retaining a principal-specific `AuthorizationPolicy`.
8. Distinguish RBAC denial from handshake failure.
9. Introduce controlled clock skew or a test certificate with invalid timing in a disposable environment.
10. Capture Envoy logs, `/certs`, proxy configuration, and a packet trace.
11. Write a rollback plan and automated rollout gates.

### Success criteria

The student must identify each failure without relying on broad restarts or globally disabling mTLS.

---

## 17. Principal-level interview answer

> I would first stop the rollout and build a directional failure matrix across source, destination, proxy revision, cluster, service account, and connection age. mTLS is a two-sided protocol, so I would avoid assuming the new caller is at fault merely because the error appears there. I would compare healthy and unhealthy proxies using proxy-status, LDS/CDS/RDS/EDS, SDS secrets, Envoy certificate state, response flags, and policy objects. Then I would classify the fault as transport reachability, TLS-mode mismatch, certificate delivery, trust-chain mismatch, identity/SAN mismatch, authorization denial, or mixed-version incompatibility.
>
> I would preserve existing healthy connections and avoid fleet-wide restarts. Recovery would use the narrowest reversible action: remove the broken destination revision, roll back the proxy/control-plane canary, restore overlapping trust, or correct the specific policy. I would use permissive mode only as a scoped, time-limited emergency bridge with monitoring and an explicit return to strict mode.
>
> Before resuming, I would validate old-to-old, old-to-new, new-to-old, and new-to-new paths, including new connections, cross-cluster calls, and the actual authenticated principal. Long term, I would gate mesh rollouts on xDS/SDS acceptance, handshake success, certificate health, authorization behavior, and synthetic compatibility tests rather than pod readiness alone.

---

## 18. Common interview traps

### “The pods are Ready, so the mesh is healthy.”

Readiness may not validate xDS acceptance, SDS delivery, cross-workload trust, or new TLS connections.

### “Just restart the sidecars.”

This can destroy healthy long-lived connections and trigger a synchronized handshake storm.

### “Set everything to permissive.”

That may restore traffic while silently removing the security property the mesh was intended to provide.

### “The certificate exists, so certificate delivery works.”

A CA can issue a certificate that never reaches or is never accepted by Envoy.

### “The error is 503, therefore it is an application outage.”

The application may never receive the request. Envoy can generate the 503 locally.

### “TLS succeeded, so identity policy is correct.”

Authentication and authorization are separate. A valid but unexpected principal can still be denied.

---

## 19. Final engineering principles

1. Treat mesh upgrades as distributed protocol migrations, not sidecar image updates.
2. Debug mTLS from both ends of the connection.
3. Separate issuance, delivery, trust, authentication, authorization, and routing.
4. Preserve healthy existing connections during diagnosis.
5. Use comparison cohorts and directional matrices.
6. Make CA rotation overlapping and measurable.
7. Do not confuse availability recovery with security recovery.
8. Gate rollout on real connection establishment and identity validation, not Kubernetes readiness alone.
9. Keep emergency downgrades narrow, expiring, observable, and reversible.
10. Design the mesh so a local identity or policy error cannot become a fleet-wide outage.
