# Principal Platform Engineering Interview Curriculum

## Chapter 3 — Multi-Cloud Routing, Workload Identity, and Secrets Architecture

> **Original scenario:** Design secure connectivity, workload identity, and secret delivery across AWS, Azure, and Google Cloud. The solution must survive partial inter-cloud failure, avoid long-lived credentials, prevent transitive route exposure, and preserve regional service when a central dependency is unavailable.

---

# 1. Why this question exists

The interviewer is not asking whether you know the names of three cloud networking products.

They are testing whether you can:

- Treat every cloud as an independent administrative and failure domain.
- Separate user ingress, service-to-service traffic, data replication, control traffic, and management access.
- Explain routing, identity, and secrets as three different systems.
- Avoid converting multi-cloud into one enormous flat network.
- Design for latency, packet loss, asymmetric routing, route leaks, and partial interconnect failure.
- Replace stored access keys with short-lived workload identity.
- Preserve local operation when a central identity or secret service is unavailable.
- Define trust boundaries, failover semantics, and data consistency.
- Make the design operable by real teams rather than theoretically portable.

A weak answer says:

> Connect the clouds with VPNs, use Vault, and synchronize secrets.

A senior answer discusses transit hubs, workload identity federation, private DNS, and regional secret stores.

A Principal-level answer starts with invariants, makes every cross-cloud dependency explicit, limits network and identity transitivity, and proves that loss of one cloud or interconnect does not cause global authentication, routing, or secret-delivery failure.

---

# 2. Interview answer first

## 2.1 Ninety-second answer

> I would not pretend the three clouds are one failure domain. Each cloud and region remains independently routable, independently authenticated, and capable of serving its critical local path when the inter-cloud connection or central control plane is impaired.
>
> I would classify traffic first. User ingress uses an existing global edge or CDN. Service-to-service traffic normally remains local and crosses clouds only through explicit gateways. Replication and backup use dedicated paths and bandwidth controls. Management access uses a separate identity and network plane. The transit architecture uses non-overlapping address space, redundant BGP sessions, prefix allowlists, maximum-prefix protection, and route ownership. I would prefer service-level exposure through gateways or private endpoints over arbitrary pod-CIDR reachability.
>
> For identity, every workload uses the native cloud mechanism: EKS Pod Identity or IRSA in AWS, Microsoft Entra Workload ID in AKS, and Workload Identity Federation for GKE. Cross-cloud access uses narrowly scoped token exchange with issuer, audience, subject, namespace, service-account, environment, and target-role restrictions. No static cloud keys are stored in Kubernetes.
>
> For secrets, I separate governance authority from serving path. A central authority may define ownership and rotation, but critical workloads retrieve or receive secrets from a regional service so a WAN or central-Vault outage does not stop local traffic. Dynamic credentials are preferred. Synchronization is one-way, versioned, observable, and revocable. I test stale-secret behavior, missed rotation, clock skew, trust-bundle change, and complete interconnect loss.
>
> The success criterion is independent survivability: one cloud, one region, or the inter-cloud network can fail without granting broader access, leaking routes, corrupting replicated data, or preventing the remaining regions from authenticating and serving their critical customer journey.

## 2.2 Fifteen-second summary

> Keep traffic local, expose cross-cloud services through explicit gateways, use native short-lived workload identity, serve secrets regionally, and ensure that loss of the interconnect cannot become a global outage or security bypass.

---

# 3. State assumptions and invariants

Use an opening such as:

> I will assume three cloud providers, multiple regions, private Kubernetes clusters, customer-facing APIs, regulated data, and a requirement that a region continue its critical path for at least several hours without the cross-cloud link or central secret authority. Synchronous cross-cloud calls are exceptions, not the default design.

Clarify:

- Active-active or active-passive service model.
- Required RTO and RPO.
- Data residency constraints.
- Whether one public endpoint already exists.
- Existing carrier, colo, SD-WAN, or cloud backbone.
- Expected bandwidth and packet rate.
- Percentage of calls that truly need another cloud.
- Whether pod CIDRs overlap.
- Who owns DNS and certificate trust.
- Whether clients can support regional fallback.
- Secret types: static, dynamic, certificate, signing key, API token.
- Maximum acceptable secret staleness.
- Required audit retention.

Core invariants:

1. A remote-cloud outage cannot exhaust local threads, sockets, or retry budgets.
2. No cloud receives broad trust merely because it is connected to the transit network.
3. No long-lived cloud credentials are embedded in images, Kubernetes Secrets, or CI variables.
4. A compromised workload can assume only the role bound to its exact workload identity.
5. Route advertisements are explicit and bounded.
6. Secret rotation is versioned and reversible.
7. One region can operate from locally available identity and secret material for the agreed isolation period.
8. Write ownership and conflict semantics are defined before traffic failover.

---

# 4. Separate the planes

```text
                     CUSTOMER TRAFFIC
Client -> Global edge/CDN -> Regional ingress -> Local services

                     SERVICE TRAFFIC
Local service -> local dependency
Local service -> explicit cross-cloud gateway -> remote service

                     DATA PLANE
Database replication / object replication / event replication

                     CONTROL PLANE
GitOps, policy, service catalog, certificate and identity governance

                     MANAGEMENT PLANE
Human access, break-glass, support, audit collection
```

Do not send all five categories through one undifferentiated transit path.

Different planes need different:

- Bandwidth and QoS.
- Authentication.
- Route policy.
- Monitoring.
- Availability target.
- Failure behavior.
- Change ownership.

---

# 5. Network architecture from first principles

## 5.1 Address planning

A multi-cloud network fails early when address ownership is unclear.

Maintain centrally governed IPAM for:

- Cloud VPC/VNet ranges.
- Kubernetes node ranges.
- Pod and service ranges.
- Transit links.
- Private endpoint ranges.
- On-premises and partner networks.

Overlapping pod networks do not automatically make multi-cloud impossible, but they prevent simple route propagation. Options include:

- Service gateways and proxies.
- NAT at the domain boundary.
- Readdressing during migration.
- Cluster-aware service networking.
- Provider-independent service VIPs.

Principal rule:

> Prefer service identity over global pod-IP identity. Pod CIDRs are implementation details and should not automatically become enterprise-wide routes.

---

## 5.2 Transit choices

Possible transport layers:

- Provider private connectivity through a colocation facility.
- Carrier-managed WAN or SD-WAN.
- Cloud WAN/transit products.
- Redundant IPsec tunnels.
- A provider-independent backbone.
- Service-level gateways over the public Internet with mTLS.

Evaluate each using:

- Failure independence.
- Bandwidth and latency.
- Encryption.
- BGP control.
- Operational ownership.
- Provisioning time.
- Cross-AZ and egress cost.
- DDoS exposure.
- Troubleshooting visibility.

Do not claim that a private circuit is automatically encrypted. Transport privacy and cryptographic protection are separate requirements.

---

## 5.3 BGP and route controls

Every BGP adjacency needs:

- Explicit allowed prefixes.
- Maximum-prefix limits.
- Route ownership.
- Local preference and failover policy.
- Community strategy where supported.
- Monitoring for unexpected advertisements.
- Symmetric return-path validation.
- Maintenance and rollback procedure.

Example policy intent:

```text
AWS advertises only:
  10.20.0.0/16 regional service networks

Azure advertises only:
  10.40.0.0/16 regional service networks

GCP advertises only:
  10.60.0.0/16 regional service networks

Never accept:
  0.0.0.0/0
  corporate user networks
  unrelated environment ranges
  another cloud's transit-learned prefixes unless explicitly approved
```

A common failure is unintended transitive routing:

```text
AWS learns on-premises routes from Azure
Azure learns GCP routes from AWS
A route leak turns one hub into an enterprise-wide transit domain
```

Prevent this through import/export policies, route tables, segmentation, and tests.

---

## 5.4 Service-level connectivity

Rather than full network reachability, expose a small number of cross-cloud services through:

- Regional API gateways.
- East-west gateways.
- Private load balancers.
- Private service-connectivity products.
- mTLS proxies.
- Event or data-replication interfaces.

```text
AWS service
   |
   v
AWS cross-cloud gateway
   |  mTLS + workload identity + rate limit
   v
Azure regional gateway
   |
   v
Azure service
```

Benefits:

- Clear authorization boundary.
- Stable service address.
- Rate limiting and circuit breaking.
- Better auditing.
- No arbitrary remote pod reachability.
- Easier protocol and version control.

Cost:

- Extra hop.
- Gateway capacity.
- Certificate and policy management.
- Potential central bottleneck if not regionalized.

---

## 5.5 DNS

Decide who resolves:

- Cloud-private service names.
- Kubernetes names.
- Cross-cloud service names.
- Public names.
- Failover names.

Use conditional forwarding or authoritative zones deliberately. Avoid circular resolver dependencies.

Failure questions:

- What happens if the central DNS forwarder is unreachable?
- Are cached records sufficient for the isolation period?
- Does negative caching extend an outage?
- Does a local resolver need the inter-cloud path to answer local names?
- Are split-horizon answers consistent with certificate identities?

DNS is not a substitute for service health or data readiness.

---

# 6. Remote-call failure model

A synchronous cross-cloud call includes more failure states than success or failure:

- Slow connection establishment.
- Packet loss.
- One-way routing.
- TLS handshake delay.
- Remote gateway healthy but dependency saturated.
- Partial response.
- Duplicate response after retry.
- Stale DNS.
- Identity token exchange failure.
- Remote success after local timeout.

Use:

- Strict deadlines.
- Small connection timeout.
- Bounded concurrency.
- Circuit breakers.
- Retry budgets.
- Jitter.
- Idempotency keys.
- Queue-based asynchronous integration where possible.
- Local cache or degraded response.

```text
Overall request deadline: 800 ms
  local application budget: 150 ms
  remote call budget: 300 ms
  one retry only when idempotent and budget remains
  fallback budget: 100 ms
```

Do not stack retries at client, gateway, sidecar, SDK, and application layers.

---

# 7. Workload identity foundations

## 7.1 Identity chain

```text
Pod
  |
  | projected Kubernetes service-account token
  v
Cloud identity federation / token service
  |
  | short-lived cloud credentials or access token
  v
Cloud API or target role
```

Security depends on validating:

- Issuer.
- Audience.
- Subject.
- Namespace.
- Service account.
- Cluster or trust domain.
- Environment.
- Target role.
- Session duration.

The Kubernetes service account is not automatically a cloud identity. A trust relationship maps or exchanges it into a cloud principal.

---

## 7.2 AWS

Use EKS Pod Identity or IRSA according to platform constraints.

EKS Pod Identity uses:

- A Pod Identity association.
- The EKS Pod Identity Agent on nodes.
- Supported AWS SDK credential-provider behavior.
- Temporary credentials.

IRSA uses:

- Cluster OIDC issuer.
- IAM role trust policy.
- Projected service-account token.
- STS web-identity exchange.

Risks to test:

- Application uses static credentials earlier in the SDK provider chain.
- Node role remains reachable and broader than workload role.
- Trust policy accepts a wildcard namespace or service account.
- Token audience or issuer mismatch.
- Proxy configuration blocks Pod Identity agent endpoints.
- Old SDK lacks required support.

---

## 7.3 Azure

Microsoft Entra Workload ID uses:

- AKS OIDC issuer.
- Projected service-account token.
- Federated identity credential.
- User-assigned managed identity or application registration.
- Azure Identity/MSAL support.

Restrict the federated subject:

```text
system:serviceaccount:payments:ledger-api
```

Test:

- Issuer mismatch after cluster replacement.
- Excessively broad role assignment.
- Multiple environments sharing one identity.
- SDK compatibility.
- Key Vault or resource firewall path.
- Token refresh during identity-provider degradation.

---

## 7.4 Google Cloud

Workload Identity Federation for GKE lets Kubernetes principals access Google Cloud APIs without service-account key files.

A principal may be represented using namespace and service-account identity in the workload identity pool.

Test:

- Direct principal binding versus IAM service-account impersonation.
- Metadata server availability.
- Namespace/subject mapping.
- Quotas and token refresh.
- Cross-project permissions.
- Legacy credential files overriding federation.

---

# 8. Cross-cloud identity federation

Example: an AWS workload needs one Azure API.

Do not copy an Azure client secret into AWS.

Possible pattern:

```text
AWS workload identity
   |
   | signed workload assertion / OIDC token
   v
Microsoft Entra federated trust
   |
   | short-lived access token
   v
Azure resource
```

Controls:

- Dedicated trust per environment.
- Exact issuer.
- Exact subject.
- Exact audience.
- Minimal target permission.
- Short lifetime.
- Session and token audit.
- Revocation process.
- Rate limits.
- No fallback to stored secret.

Cross-cloud federation is powerful because it removes keys, but it creates a trust bridge. A compromised issuer or broad subject expression can become a cloud-to-cloud escalation path.

---

# 9. Human identity

Human access is separate from workload identity.

Use:

- Corporate identity provider.
- MFA.
- Device and risk policy.
- Just-in-time elevation.
- Privileged-access workflow.
- Short sessions.
- Separate production roles.
- Central audit.

Break-glass accounts:

- Stored and protected independently.
- Hardware-backed MFA.
- No routine use.
- Immediate alert on access.
- Tested periodically.
- Credentials rotated after use.

Do not give humans the workload role and do not let workloads impersonate broad human administrator roles.

---

# 10. Secrets architecture

## 10.1 Secret categories

| Type | Preferred pattern |
|---|---|
| Cloud API access | Workload identity, not a secret |
| Database password | Dynamic credential where supported |
| TLS workload identity | Automated short-lived certificate |
| Third-party static token | Regional encrypted copy with rotation |
| Signing key | HSM/KMS operation rather than key export |
| Bootstrap credential | Narrow, short-lived, one-time use |

The first design question is whether the value should exist as a portable secret at all.

---

## 10.2 Runtime retrieval

```text
Pod authenticates to regional Vault/secret store
 -> receives short-lived secret
 -> renews lease
 -> revokes on termination or compromise
```

Advantages:

- Fewer persistent copies.
- Dynamic credentials.
- Central audit.
- Rapid revocation.

Risks:

- Runtime dependency.
- Token-renewal failure.
- Cold-start latency.
- Secret-store saturation.
- Central outage.

Mitigations:

- Regional clusters or stores.
- Agent caching with bounded lifetime.
- Lease-renewal alerts.
- Stale-secret policy.
- Capacity testing.

---

## 10.3 Controlled synchronization

A central governance source may synchronize selected values one-way to regional stores.

```text
Central authority
   |
   +--> AWS Secrets Manager regional copy
   +--> Azure Key Vault regional copy
   +--> Google Secret Manager regional access path
```

Advantages:

- Local availability.
- Native cloud integration.
- Lower runtime WAN dependency.

Risks:

- More encrypted copies.
- Version divergence.
- Deletion and revocation complexity.
- Destination-specific permissions.
- Sync lag.

Required metadata:

- Secret owner.
- Classification.
- Source version.
- Destination version.
- Rotation deadline.
- Consumers.
- Last successful sync.
- Expiry.
- Revocation status.

Vault Secrets Sync is one example of one-way synchronization; it should not be treated as bidirectional configuration merging.

---

## 10.4 Kubernetes delivery choices

### Native Kubernetes Secret

Advantages:

- Familiar.
- Works with environment variables and volumes.
- Easy controller integration.

Risks:

- Stored in etcd.
- Often copied into manifests, logs, or support output.
- Environment variables do not update automatically.

### CSI-mounted file

Advantages:

- Secret can be mounted without normal application API access.
- Rotation can update files.
- Avoids some environment-variable problems.

Risks:

- Application must reload.
- Node plugin becomes part of availability path.
- Mount failure affects pod startup.

### Sidecar or agent

Advantages:

- Authentication, renewal, templates, and local cache.

Risks:

- Additional process and lifecycle coupling.
- File permissions and reload behavior.

### Direct application retrieval

Advantages:

- Full control over refresh and dynamic credentials.

Risks:

- Every application must implement correct retry, cache, and revocation behavior.

Choose deliberately by secret type and failure model.

---

# 11. Certificate and trust design

Define:

- Root trust ownership.
- Intermediate CAs per cloud/region/domain.
- Certificate lifetime.
- Rotation overlap.
- Revocation or short-lifetime strategy.
- Trust-domain federation.
- SAN naming.
- Clock synchronization.

Avoid one online global CA as the only issuance path for all workloads.

A common model:

```text
Offline or tightly controlled root
  +--> AWS regional intermediate
  +--> Azure regional intermediate
  +--> GCP regional intermediate
```

Federate only required trust domains. A certificate that authenticates a workload does not automatically authorize it.

---

# 12. Data architecture

Routing a request to another cloud does not make the data ready.

Clarify:

- Source of truth.
- Replication direction.
- Maximum lag.
- Conflict resolution.
- Write ownership.
- Schema compatibility.
- Queue replay.
- Duplicate processing.
- Failback.

Patterns:

### Single writer

- Simpler consistency.
- Remote regions may depend on writer availability.
- Failover requires fencing old writer.

### Multi-writer

- Better local write availability.
- Requires conflict semantics, idempotency, clock and ordering assumptions.

### Local write with asynchronous reconciliation

- Useful where business operation tolerates eventual consistency.
- Must communicate degraded state and conflict outcome.

Principal statement:

> The network can move traffic in seconds; state ownership may take longer. RTO must include safe data and dependency transition, not only edge routing.

---

# 13. Failure-mode analysis

## 13.1 Inter-cloud link loss

Expected behavior:

- Local critical path remains local.
- Remote circuit opens quickly.
- Retries remain bounded.
- Replication backlog grows visibly.
- Noncritical cross-cloud feature degrades.
- No route oscillation.

## 13.2 Slow link

Often worse than complete loss.

Risks:

- Threads wait.
- Connections accumulate.
- Retry traffic multiplies.
- Health checks continue passing.
- Replication competes with customer traffic.

Use latency-based circuit breaking, deadlines, concurrency limits, and separate replication bandwidth.

## 13.3 Identity-provider outage

Test:

- Existing token lifetime.
- Refresh behavior.
- New pod startup.
- Cached JWKS behavior.
- Regional federation endpoints.
- Degraded local operation.

Do not extend token lifetimes without evaluating compromise exposure.

## 13.4 Secret sync misses one region

Detect through source/destination version metrics. Decide whether workload:

- Continues with old secret.
- Stops new traffic.
- Uses overlapping credential versions.
- Fails closed for a sensitive operation.

## 13.5 Route leak

Immediate response:

- Stop advertisement.
- Remove imported route.
- Freeze transit changes.
- Confirm no unauthorized traffic path.
- Review BGP and firewall logs.
- Rotate credentials if exposure occurred.

## 13.6 Cloud compromise

Connectivity must not imply trust. Containment may include:

- Revoke federation trust.
- Withdraw specific routes.
- Disable cross-cloud gateway identity.
- Fence replication.
- Preserve unaffected cloud operation.

---

# 14. Observability

Network signals:

- BGP session state.
- Prefix count and unexpected prefixes.
- Tunnel status.
- Packet loss and latency.
- Throughput and drops.
- Asymmetric-flow evidence.
- Gateway saturation.

Identity signals:

- Token exchange success and latency.
- Role-assumption count.
- Subject, audience, and issuer failures.
- Use of legacy/static credentials.
- Unusual role chaining.
- Token age and refresh failures.

Secret signals:

- Version divergence.
- Sync lag.
- Lease-renewal failure.
- Expiry horizon.
- Retrieval latency.
- Regional store availability.
- Unauthorized access attempts.

Product signals:

- Local user-journey success.
- Degraded-mode activation.
- Cross-cloud dependency error rate.
- Replication backlog age.
- Regional write availability.

---

# 15. Investigation commands and evidence

AWS identity:

```bash
aws eks list-pod-identity-associations --cluster-name <cluster>
aws eks describe-pod-identity-association \
  --cluster-name <cluster> --association-id <id>
aws sts get-caller-identity
```

Azure identity:

```bash
az aks show -g <rg> -n <cluster> --query oidcIssuerProfile
az identity federated-credential list \
  --identity-name <identity> --resource-group <rg>
az role assignment list --assignee <principal-id>
```

GCP identity:

```bash
gcloud container clusters describe <cluster> \
  --region <region> --format='value(workloadIdentityConfig.workloadPool)'
gcloud projects get-iam-policy <project>
```

Kubernetes token claims in a lab:

```bash
TOKEN=$(kubectl create token ledger-api -n payments --audience=<audience>)
python - <<'PY'
import os, json, base64
p=os.environ['TOKEN'].split('.')[1]
p += '=' * (-len(p) % 4)
print(json.dumps(json.loads(base64.urlsafe_b64decode(p)), indent=2))
PY
```

Routing evidence depends on transit implementation, but always collect:

- Effective routes.
- Advertised and received prefixes.
- BGP neighbor state.
- Flow logs.
- Gateway connection and error metrics.
- Packet captures at both sides when permitted.

---

# 16. Rollout strategy

1. Inventory flows and classify planes.
2. Build central IPAM and route ownership.
3. Establish redundant transit with only test prefixes.
4. Deploy regional gateways.
5. Migrate one idempotent low-risk service.
6. Introduce native workload identity.
7. Prove static credentials are no longer used.
8. Introduce regional secret delivery.
9. Test interconnect loss and identity/secret isolation.
10. Migrate critical services only after acceptance criteria pass.

Canary dimensions:

- One environment.
- One region.
- One namespace.
- One service account.
- One cross-cloud API.
- One secret type.

Rollback must remove trust and routes cleanly, not leave a hidden fallback credential.

---

# 17. Acceptance criteria

Examples:

- No static cloud keys detected in cluster or CI scanning.
- Every production workload role maps to exact namespace and service account.
- Unexpected route advertisements are rejected automatically.
- Cross-cloud gateway remains below defined utilization under failover load.
- Local critical journey remains within SLO during complete interconnect loss.
- Remote-call concurrency remains bounded during a slow-link experiment.
- Secret version divergence is detected within the agreed interval.
- Regional secret store sustains cold-start and rotation load.
- Federation revocation takes effect within the documented time.
- Replication backlog and recovery meet RPO/RTO.
- Failback is tested, not assumed.

---

# 18. Dangerous answers

## “Make one flat network across all clouds.”

This expands route, security, and failure blast radius and makes ownership ambiguous.

## “Private connectivity means secure connectivity.”

Private routing does not provide workload identity, authorization, or necessarily encryption.

## “Vault is the single source, so every workload should call one global cluster.”

Governance can be centralized while serving remains regional. A global runtime dependency can create global startup and rotation failure.

## “Synchronize all secrets everywhere.”

This multiplies exposure and makes revocation harder. Synchronize only required values.

## “Use one role for the namespace.”

Namespace-level grouping may be appropriate in limited cases, but high-value workloads need exact subject and least-privilege bindings.

## “Cross-cloud retries make it reliable.”

Retries can amplify congestion and duplicate side effects. Use budgets, deadlines, and idempotency.

## “Traffic failover solves disaster recovery.”

It does not solve state, capacity, dependency, write ownership, or failback.

---

# 19. Trade-off table

| Decision | Benefit | Risk |
|---|---|---|
| Full routed connectivity | Flexible | Large attack and route-leak surface |
| Service gateways | Explicit boundary | Extra hop and capacity domain |
| Central secret retrieval | Fewer copies | Runtime global dependency |
| Regional secret copies | Local survivability | Version divergence and more copies |
| Native identity per cloud | Strong integration | Different operational models |
| One abstraction layer | Consistent developer API | Lowest-common-denominator and hidden behavior |
| Synchronous remote call | Immediate response | Latency and correlated failure |
| Event-driven integration | Decoupling | Eventual consistency and replay complexity |
| Single writer | Simpler consistency | Failover fencing required |
| Multi-writer | Local availability | Conflict and ordering complexity |

---

# 20. Adversarial follow-ups

### Why not route pod CIDRs directly?

Because pod addresses are ephemeral implementation details and may overlap. I expose stable service interfaces and advertise only necessary domain prefixes.

### What if the central identity provider is down?

Existing short-lived tokens continue until expiry; new issuance may fail. I define the isolation window, regional dependencies, cache behavior, and safe degraded mode rather than silently extending credentials.

### How do you revoke one compromised workload?

Remove or deny its exact federation binding, revoke or disable its target role, stop the workload, and inspect token and API audit logs. Broad namespace roles make this harder.

### How do you rotate a third-party token without outage?

Use overlapping versions when supported, distribute new version, verify consumers, switch traffic, revoke old version, and alert on any old-version use.

### Would you standardize on Vault everywhere?

I may standardize governance and dynamic-secret patterns, but I preserve regional serving and use cloud-native identity where it is strongest. Standardization must not create a global runtime dependency.

### How do you prevent the remote cloud from becoming a retry sink?

Concurrency limits, circuit breaking, deadlines, retry budgets, local fallback, and product-level degradation.

---

# 21. Hands-on lab

## Objective

Prove short-lived identity and controlled cross-cloud access without stored keys.

## Lab outline

1. Create a Kubernetes service account in one cluster.
2. Enable the cloud's native workload identity.
3. Bind one read-only role to the exact service account.
4. Verify access from the correct pod.
5. Verify denial from another namespace and service account.
6. Inspect token claims.
7. Remove the binding and measure revocation behavior.
8. Introduce a regional secret and version rotation.
9. Block the interconnect or central secret endpoint in a lab.
10. Verify local service behavior and monitoring.

Lab report:

- Which token issuer and audience were used?
- Which claim selected the workload?
- Could the pod reach node credentials?
- What happened when token refresh failed?
- How long could the workload continue?
- Was the secret cached?
- How was old-version use detected?

---

# 22. Personal story mapping

Prepare a story involving:

- Cloud network transit or VPN/BGP failure.
- IAM federation or credential migration.
- Secret rotation.
- Cross-region dependency failure.
- A route, DNS, or certificate issue.

Use:

```text
Situation
Constraint
Architecture
Failure model
Decision
Evidence
Mitigation
Result
Prevention
Trade-off
```

Strong connection:

> The core principle was not multi-cloud for its own sake. It was removing hidden shared dependencies and ensuring each failure domain could be isolated without losing identity or control.

---

# 23. Memorization card

```text
Separate traffic planes
 -> keep normal path local
 -> explicit gateways for cross-cloud calls
 -> bounded BGP prefixes and no transitive leaks
 -> native workload identity in each cloud
 -> exact issuer/audience/subject trust
 -> no static keys
 -> governance can be central, serving must be regional
 -> dynamic secrets where possible
 -> test slow link, identity outage, missed rotation, and failback
 -> routing never proves state readiness
```

Final Principal sentence:

> I design multi-cloud as cooperating independent systems, not one flat platform: traffic, identity, and secrets cross boundaries only through explicit contracts, and every region retains enough local authority to survive the loss of another cloud or the interconnect without weakening security.

---

# 24. Official primary references

- Amazon EKS Pod Identity:  
  https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html
- How EKS Pod Identity works:  
  https://docs.aws.amazon.com/eks/latest/userguide/pod-id-how-it-works.html
- Microsoft Entra Workload ID for AKS:  
  https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview
- Workload Identity Federation for GKE:  
  https://cloud.google.com/kubernetes-engine/docs/concepts/workload-identity
- Google Cloud Workload Identity Federation:  
  https://cloud.google.com/iam/docs/workload-identity-federation
- Vault Kubernetes authentication:  
  https://developer.hashicorp.com/vault/docs/auth/kubernetes
- Vault Secrets Sync:  
  https://developer.hashicorp.com/vault/docs/sync
- Kubernetes service-account tokens:  
  https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/
