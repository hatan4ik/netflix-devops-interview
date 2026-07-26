# Chapter 7 — DNS Outage in a Service Mesh

## Original interview question

A large Kubernetes environment uses a service mesh. During a production incident, applications begin failing with intermittent name-resolution errors. Some requests continue to succeed because existing connections remain open, while new connections fail. CoreDNS appears healthy, sidecars are running, and only a subset of nodes is affected. How would you investigate, mitigate, and permanently fix the incident?

---

## What the interviewer is testing

This question tests whether you can reason across layers rather than treating DNS as one component.

A strong answer must separate:

- application resolver behavior;
- pod `/etc/resolv.conf` configuration;
- kubelet DNS policy;
- CoreDNS service reachability;
- CoreDNS process health and upstream recursion;
- NodeLocal DNSCache behavior;
- CNI, kube-proxy, iptables, IPVS, or eBPF datapath behavior;
- service-mesh DNS capture and proxying;
- Envoy cluster discovery and connection reuse;
- negative caching, TTLs, retries, and thundering herds;
- node-local versus cluster-wide failure domains.

The interviewer also wants to see whether you can preserve traffic while debugging, avoid restart storms, and identify the smallest safe mitigation.

---

## Foundations

### DNS is a request path, not a single server

A typical pod lookup can traverse this path:

```text
application resolver
        |
        v
pod /etc/resolv.conf
        |
        v
mesh DNS capture or node-local listener
        |
        v
NodeLocal DNSCache or kube-dns ClusterIP
        |
        v
CoreDNS pods
        |
        +--> Kubernetes API/service records
        |
        +--> upstream recursive resolver
```

Any layer can fail independently.

### Existing connections hide DNS outages

DNS is commonly used during connection establishment. Once a client has an established HTTP/2, gRPC, database, or keep-alive connection, traffic may continue even while new lookups fail.

That creates a misleading symptom pattern:

- existing streams succeed;
- newly scaled pods fail;
- restarted clients fail immediately;
- some hosts appear healthy because their connection pools are warm;
- incident impact grows as connections expire.

### Service mesh does not eliminate DNS

A sidecar may receive service endpoints from the mesh control plane, but applications still commonly resolve names before opening a socket. Depending on mesh configuration, DNS may be:

- resolved directly by the application;
- captured by the sidecar;
- answered from mesh registry data;
- forwarded to the original resolver;
- synthesized for service entries or virtual IPs.

The exact path must be proven from configuration and packet flow.

---

## Assumptions

For the interview, state assumptions explicitly:

- Kubernetes runs multiple CoreDNS replicas behind the `kube-dns` Service.
- Pods use `ClusterFirst` unless otherwise specified.
- A service mesh such as Istio injects Envoy sidecars.
- NodeLocal DNSCache may or may not be enabled.
- Internal service names and external domains are both affected until proven otherwise.
- The failure is intermittent and concentrated on a subset of nodes.
- Existing connections are masking part of the blast radius.

---

## Interview-ready answer

I would first determine whether the failure is application-specific, pod-specific, node-specific, or cluster-wide. I would compare successful and failing pods and test three paths separately: direct lookup against the pod-configured resolver, direct lookup against the `kube-dns` Service IP, and direct lookup against a CoreDNS pod IP. That isolates resolver configuration, service routing, and CoreDNS itself.

Because only some nodes are affected, I would immediately suspect a node-local datapath or cache issue: NodeLocal DNSCache, kube-proxy rules, eBPF service translation, conntrack exhaustion, CNI policy, or mesh DNS capture. I would inspect `/etc/resolv.conf`, DNS capture settings, listening sockets on port 53, packet traces, conntrack state, and per-node DNS error metrics.

For mitigation, I would avoid restarting all clients because that destroys warm connections and increases DNS demand. I would cordon affected nodes, preserve healthy connections, shift new workloads to healthy nodes, scale CoreDNS if it is saturated, and disable or bypass a broken node-local or mesh DNS layer only in a controlled canary. The permanent fix would address the failed layer, add node-scoped DNS SLOs and synthetic probes, bound retries, use caching with safe TTLs, and continuously test resolution from every node and traffic path.

---

## Components and mechanics

## 1. Application resolver behavior

Applications do not all resolve names the same way.

Important differences include:

- glibc versus musl behavior;
- JVM DNS caching;
- Go resolver modes;
- language-specific connection pools;
- search-domain expansion;
- `ndots` behavior;
- negative caching;
- client retry policy.

A lookup such as:

```text
payments
```

may generate multiple queries:

```text
payments.namespace.svc.cluster.local
payments.svc.cluster.local
payments.cluster.local
payments.<search-domain>
payments
```

High `ndots` values and long search lists can multiply query volume and latency.

### Debugging commands

```bash
cat /etc/resolv.conf
getent hosts payments
nslookup payments
nslookup payments.namespace.svc.cluster.local
dig +search payments
dig +trace example.com
```

Use both short and fully qualified names. If the FQDN works but the short name fails, inspect search domains and `ndots`.

---

## 2. Pod DNS policy and kubelet configuration

Kubernetes generates pod resolver configuration from:

- `dnsPolicy`;
- `dnsConfig`;
- kubelet `clusterDNS`;
- node resolver settings;
- host-network behavior.

Common policies:

- `ClusterFirst`;
- `Default`;
- `ClusterFirstWithHostNet`;
- `None`.

Failure patterns include:

- wrong cluster DNS IP;
- stale kubelet configuration;
- pods using host DNS unexpectedly;
- host-networked workloads missing `ClusterFirstWithHostNet`;
- oversized search lists;
- mixed node pools with different resolver configuration.

Compare a failing pod and a healthy pod byte-for-byte:

```bash
kubectl exec healthy-pod -- cat /etc/resolv.conf
kubectl exec failing-pod -- cat /etc/resolv.conf
```

---

## 3. CoreDNS Service path

The `kube-dns` Service is usually a stable virtual IP fronting CoreDNS endpoints.

The request path depends on the cluster dataplane:

- iptables;
- IPVS;
- eBPF service translation;
- NodeLocal DNSCache interception.

A healthy CoreDNS pod does not prove that the Service IP is reachable.

Test in layers:

```bash
# Resolver configured inside the pod
nslookup kubernetes.default.svc.cluster.local

# Query kube-dns Service IP directly
dig @<kube-dns-service-ip> kubernetes.default.svc.cluster.local

# Query a specific CoreDNS pod directly
dig @<coredns-pod-ip> kubernetes.default.svc.cluster.local
```

Interpretation:

| Test | Result | Likely fault |
|---|---|---|
| Pod resolver fails, Service IP works | bad pod resolver or mesh capture |
| Service IP fails, CoreDNS pod IP works | service datapath, kube-proxy, eBPF, conntrack |
| CoreDNS pod IP fails | CoreDNS process, network policy, pod networking |
| Internal works, external fails | upstream recursion, forward plugin, egress |
| External works, internal fails | Kubernetes plugin, API access, stale records |

---

## 4. CoreDNS internals

CoreDNS can be running yet degraded.

Inspect:

- CPU throttling;
- memory pressure;
- request concurrency;
- upstream timeouts;
- SERVFAIL rate;
- cache hit rate;
- loop detection;
- Kubernetes API connectivity;
- plugin latency;
- readiness and lameduck behavior.

Useful signals:

```text
coredns_dns_requests_total
coredns_dns_responses_total
coredns_dns_request_duration_seconds
coredns_cache_hits_total
coredns_cache_misses_total
coredns_forward_request_duration_seconds
coredns_forward_healthcheck_failures_total
```

Logs may show:

- `i/o timeout`;
- `no route to host`;
- `SERVFAIL`;
- upstream resolver failures;
- loop plugin errors;
- Kubernetes API watch failures.

Commands:

```bash
kubectl -n kube-system get deploy coredns
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide
kubectl -n kube-system logs deploy/coredns --tail=200
kubectl -n kube-system top pods -l k8s-app=kube-dns
kubectl -n kube-system get configmap coredns -o yaml
```

---

## 5. NodeLocal DNSCache

NodeLocal DNSCache reduces cross-node DNS traffic and avoids some conntrack pressure, but it introduces a node-scoped dependency.

A typical path becomes:

```text
pod -> node-local listener -> CoreDNS -> upstream
```

Failure modes include:

- DaemonSet absent on some nodes;
- listener not bound to the expected IP;
- stale iptables rules;
- local cache process crash-looping;
- local port 53 conflict;
- host networking issue;
- wrong upstream kube-dns IP;
- cache poisoning or stale negative entries;
- node pressure causing packet loss or CPU starvation.

Validate per node:

```bash
kubectl -n kube-system get ds node-local-dns
kubectl -n kube-system get pods -l k8s-app=node-local-dns -o wide
kubectl -n kube-system logs <node-local-dns-pod>
ss -lntup | grep ':53'
```

If only pods on specific nodes fail, node-local DNS becomes a top suspect.

---

## 6. Service-mesh DNS capture

A mesh may redirect DNS traffic from the application to the sidecar.

Questions to answer:

- Is DNS capture enabled globally or by workload?
- Does Envoy answer from its registry or forward queries?
- Are synthetic VIPs used?
- Is the failing name represented in the mesh registry?
- Are ServiceEntries stale or malformed?
- Is the sidecar listening on the expected DNS port?
- Are iptables redirection rules present?

Inspect sidecar configuration and listeners:

```bash
istioctl proxy-config listeners <pod> -n <namespace>
istioctl proxy-config clusters <pod> -n <namespace>
istioctl proxy-config endpoints <pod> -n <namespace>
istioctl proxy-config bootstrap <pod> -n <namespace>
```

Compare a working and failing pod. A mesh rollout can create mixed behavior where some pods capture DNS and others do not.

A useful isolation test is to query the configured resolver directly and compare it with a query that traverses normal application behavior.

---

## 7. Conntrack exhaustion

DNS commonly uses UDP. In iptables-based service routing, UDP flows consume conntrack entries.

Symptoms can include:

- random lookup timeouts;
- node-specific failures;
- direct CoreDNS pod queries working while Service IP queries fail;
- packet drops without CoreDNS receiving traffic;
- failures during bursts, deployments, or restart storms.

Inspect:

```bash
conntrack -S
sysctl net.netfilter.nf_conntrack_count
sysctl net.netfilter.nf_conntrack_max
dmesg | grep -i conntrack
```

Do not treat increasing the conntrack limit as the only fix. Also reduce query multiplication, enable effective caching, inspect retries, and identify why the node creates excessive flows.

---

## 8. Network policy and dataplane enforcement

DNS may be blocked by:

- Kubernetes NetworkPolicy;
- CiliumNetworkPolicy;
- host firewall rules;
- security groups;
- eBPF policy maps;
- sidecar egress restrictions.

Verify whether UDP and TCP port 53 are both allowed. Large DNS responses may fall back to TCP.

A policy that allows only UDP/53 can create selective failures for large responses, DNSSEC, or truncated packets.

Packet-level inspection:

```bash
tcpdump -ni any port 53
```

With Cilium:

```bash
cilium monitor --type drop
hubble observe --protocol dns
hubble observe --from-pod <namespace>/<pod>
```

---

## 9. MTU, fragmentation, and TCP fallback

Some DNS responses exceed the safe UDP size. EDNS0, DNSSEC, and long record sets can trigger fragmentation or TCP fallback.

Failure patterns:

- small internal lookups succeed;
- large external responses fail;
- only certain domains fail;
- UDP packets leave but replies are fragmented and dropped;
- TCP/53 is blocked.

Tests:

```bash
dig example.com +notcp
dig example.com +tcp
dig example.com +bufsize=1232
```

If TCP works but UDP fails, inspect MTU and fragmentation. If UDP works but TCP fails, inspect policy and firewall rules.

---

## 10. Negative caching and stale failures

NXDOMAIN and SERVFAIL can be cached by clients, local caches, or recursive resolvers.

This means the system may remain degraded after the original fault is fixed.

During mitigation:

- understand which layer caches the error;
- avoid mass restarts unless necessary;
- use targeted cache flushes where supported;
- wait for TTL expiration when safer than disruption;
- verify with unique query names to avoid cached results.

---

## Scaling math

Assume:

- 20,000 pods;
- each pod performs 2 logical lookups per second;
- search expansion creates 4 DNS queries per logical lookup;
- a retry policy retries twice after failure.

Normal query rate:

```text
20,000 × 2 × 4 = 160,000 queries/second
```

During failure with two retries:

```text
160,000 × 3 = 480,000 queries/second
```

If a rollout restarts 25% of workloads and cold clients perform ten startup lookups each, the burst becomes much larger.

The important Staff-level point is that DNS failure can become self-amplifying:

```text
latency -> timeout -> retry -> more DNS traffic -> saturation -> more latency
```

Bound retries, add jitter, cache positive answers, and avoid synchronized restarts.

---

## Failure tree

```text
Application reports name resolution failure
|
+-- Application-specific?
|   +-- resolver cache
|   +-- search domains / ndots
|   +-- runtime-specific behavior
|
+-- Pod-specific?
|   +-- wrong resolv.conf
|   +-- mesh DNS capture mismatch
|   +-- sidecar listener failure
|
+-- Node-specific?
|   +-- NodeLocal DNSCache
|   +-- conntrack exhaustion
|   +-- kube-proxy / eBPF service path
|   +-- CNI or host firewall
|   +-- CPU, packet loss, MTU
|
+-- Cluster-wide?
|   +-- CoreDNS saturation
|   +-- bad Corefile rollout
|   +-- Kubernetes API access failure
|   +-- upstream recursive resolver outage
|
+-- Domain-specific?
    +-- internal service record problem
    +-- external recursion failure
    +-- large response / TCP fallback
    +-- stale negative cache
```

---

## Debugging workflow

## Step 1 — Define the blast radius

Capture:

- affected namespaces;
- affected nodes;
- internal versus external names;
- short names versus FQDNs;
- UDP versus TCP;
- old versus newly started pods;
- mesh-injected versus non-injected pods;
- NodeLocal DNSCache enabled versus disabled nodes.

Do not begin with a cluster-wide restart.

## Step 2 — Compare healthy and failing pods

Run the same command set from both:

```bash
cat /etc/resolv.conf
getent hosts kubernetes.default.svc.cluster.local
dig kubernetes.default.svc.cluster.local
dig example.com
```

Record latency, response code, resolver IP, and whether the query reaches CoreDNS.

## Step 3 — Bypass layers one at a time

```text
normal resolver
    -> node-local IP
    -> kube-dns Service IP
    -> CoreDNS pod IP
    -> upstream resolver
```

The first transition from failure to success identifies the broken layer.

## Step 4 — Inspect packets

On a failing node:

```bash
tcpdump -ni any port 53
```

Determine:

- whether the query leaves the pod;
- whether it reaches the node-local listener;
- whether it reaches CoreDNS;
- whether a reply returns;
- whether the reply is dropped or fragmented.

## Step 5 — Inspect node state

```bash
conntrack -S
ss -lntup | grep ':53'
iptables-save | grep -i dns
ip route
ip rule
```

For eBPF dataplanes, inspect service maps and drop events.

## Step 6 — Inspect CoreDNS

Check saturation, logs, upstream errors, and configuration rollout timing.

## Step 7 — Inspect mesh behavior

Compare sidecar versions, bootstrap config, DNS listeners, and registry state between healthy and failing pods.

## Step 8 — Correlate with changes

Look for:

- CoreDNS ConfigMap changes;
- mesh control-plane upgrades;
- sidecar version rollout;
- node image changes;
- kube-proxy or CNI upgrades;
- NetworkPolicy changes;
- upstream resolver changes;
- large deployment or autoscaling event.

---

## Immediate mitigation

Choose the smallest safe action supported by evidence.

### If specific nodes are affected

- cordon affected nodes;
- stop scheduling new workloads there;
- preserve running workloads if existing connections remain healthy;
- drain gradually after replacement capacity is available;
- restart or replace the broken node-local DNS component only on affected nodes.

### If CoreDNS is saturated

- scale replicas;
- spread them across nodes and zones;
- remove CPU throttling;
- increase resource requests appropriately;
- reduce retry amplification;
- enable or restore caching.

### If a bad CoreDNS configuration was deployed

- roll back the ConfigMap and deployment;
- validate syntax before rollout;
- canary changes to a subset of resolvers.

### If mesh DNS capture is broken

- canary-disable DNS capture for affected workloads;
- roll back the sidecar or mesh configuration;
- avoid changing every namespace at once;
- confirm that direct cluster DNS works first.

### If the Service IP path is broken

- shift workloads away from affected nodes;
- repair kube-proxy, eBPF service maps, or node networking;
- use direct resolver bypass only as a short-lived emergency measure with clear rollback.

### What not to do

Do not restart every application. That destroys healthy connection pools and creates a synchronized DNS storm.

---

## Permanent fixes

### Architecture

- run multiple CoreDNS replicas across failure domains;
- set PodDisruptionBudgets and topology spread constraints;
- use NodeLocal DNSCache when appropriate, but monitor it per node;
- ensure UDP and TCP DNS paths work;
- maintain redundant upstream resolvers;
- avoid a single shared external resolver dependency.

### Configuration safety

- validate CoreDNS configuration in CI;
- use canary rollouts;
- version resolver configuration;
- test internal and external names before promotion;
- verify mesh DNS capture compatibility during sidecar upgrades.

### Client behavior

- use FQDNs for critical in-cluster dependencies where appropriate;
- reduce unnecessary search expansion;
- bound retries;
- add exponential backoff and jitter;
- maintain connection pools;
- avoid zero-TTL or hyperactive re-resolution without a clear reason.

### Observability

Measure by node, namespace, resolver path, and response code:

- DNS success rate;
- p50, p95, and p99 lookup latency;
- NXDOMAIN and SERVFAIL rates;
- CoreDNS saturation;
- cache hit ratio;
- upstream latency;
- node-local cache availability;
- conntrack utilization;
- mesh DNS capture failures;
- TCP fallback success.

### Continuous verification

Run synthetic probes from every node pool:

- internal Service lookup;
- headless Service lookup;
- external lookup;
- UDP lookup;
- TCP lookup;
- query through mesh-injected and non-injected pods.

---

## SLO design

A DNS SLO should measure the customer-visible operation, not merely whether CoreDNS pods are running.

Example SLIs:

```text
successful DNS answers / total valid DNS queries
```

and:

```text
percentage of synthetic lookups completed below 100 ms
```

Segment by:

- node pool;
- availability zone;
- internal versus external domain;
- mesh capture enabled versus disabled;
- UDP versus TCP.

A global aggregate can hide a complete outage on one node pool.

---

## Security considerations

- restrict who can modify CoreDNS and mesh DNS configuration;
- audit ConfigMap and control-plane changes;
- prevent workloads from bypassing approved resolvers without authorization;
- enforce egress policy without blocking required DNS transport;
- monitor unexpected query destinations;
- consider DNS tunneling detection;
- validate ServiceEntry and external-name ownership;
- protect resolver logs because queries can expose sensitive service names.

---

## Wrong answers

### “CoreDNS pods are healthy, so DNS is healthy”

Wrong because Service routing, node-local caches, mesh capture, and upstream recursion may still be broken.

### “Restart all pods”

Dangerous because it destroys working connections and amplifies DNS demand.

### “Increase DNS timeout and retry count”

This may worsen saturation and prolong user-visible latency.

### “The mesh handles discovery, so DNS is irrelevant”

Incorrect for most application connection paths.

### “Allow UDP port 53 and the policy is complete”

DNS may require TCP fallback.

### “Scale CoreDNS immediately”

Scaling does not fix a node-local datapath failure and may add no value if queries never reach CoreDNS.

---

## Trade-offs

### NodeLocal DNSCache

Benefits:

- lower latency;
- less cross-node traffic;
- reduced conntrack pressure;
- local caching.

Costs:

- another node-level dependency;
- per-node inconsistency;
- harder debugging;
- DaemonSet rollout risk.

### Mesh DNS capture

Benefits:

- integration with mesh service registry;
- support for synthetic addresses;
- richer control over external service resolution.

Costs:

- more complexity;
- mixed behavior during rollout;
- coupling to sidecar configuration;
- difficult packet-path diagnosis.

### Long TTLs

Benefits:

- lower query volume;
- resilience during brief resolver outages.

Costs:

- slower endpoint change propagation;
- stale answers during failover.

### Short TTLs

Benefits:

- faster convergence.

Costs:

- higher query load;
- greater outage amplification.

---

## 90-second answer

I would begin by proving the failure domain. I would compare healthy and failing pods and test the normal resolver, the node-local resolver if present, the kube-dns Service IP, and a CoreDNS pod IP. That separates application configuration, mesh DNS capture, node-local caching, service routing, and CoreDNS itself.

Because only some nodes are affected, I would prioritize NodeLocal DNSCache, conntrack exhaustion, kube-proxy or eBPF service translation, CNI policy, and sidecar DNS interception. I would capture port-53 traffic on a failing node and determine where the query or response disappears. I would also compare mesh and resolver configuration between healthy and failing pods.

For mitigation, I would preserve existing connections, cordon affected nodes, shift new workloads to healthy capacity, and repair only the failing DNS layer. I would not restart all clients because that creates a cold-connection and DNS storm. The permanent solution would include redundant CoreDNS, node-scoped DNS monitoring, bounded retries, safe caching, UDP and TCP testing, canary configuration rollout, and synthetic resolution checks from every node pool.

---

## Deep answer structure

Use this order in a 10- to 15-minute discussion:

1. Define blast radius.
2. Explain why existing connections still work.
3. Draw the complete DNS path.
4. Bypass each layer experimentally.
5. Focus on node-local failure mechanisms.
6. Inspect packets and conntrack.
7. Inspect CoreDNS and upstream recursion.
8. Inspect mesh capture and sidecar version skew.
9. Mitigate without restart amplification.
10. Propose permanent architecture and SLOs.

---

## Whiteboard

```text
                    Kubernetes API
                         |
                         v
Application -> Envoy DNS capture? -> NodeLocal DNS -> kube-dns VIP
     |                                      |              |
     |                                      |              v
     |                                      +---------> CoreDNS
     |                                                     |
     +---- existing connection ----------------------------+
                                                           |
                                                           v
                                                 upstream resolver
```

Annotate the diagram with:

- where caching occurs;
- where retries occur;
- where conntrack is used;
- where policy is enforced;
- where packet capture should be performed;
- which layers are node-scoped versus cluster-scoped.

---

## Adversarial follow-ups

### Why do only newly started pods fail?

They have no warm DNS cache or established connection pool. Existing pods may continue using open HTTP/2 or database connections.

### Why would querying the CoreDNS pod IP work while the Service IP fails?

The virtual Service path is broken: kube-proxy, IPVS, eBPF translation, conntrack, or node-local interception.

### Why can only some domains fail?

Internal and external names use different CoreDNS plugins and upstream paths. Large responses may also require TCP fallback.

### Why can DNS failure remain after the resolver is repaired?

Negative answers or SERVFAIL results may remain cached in applications, local caches, or recursive resolvers.

### Would you bypass DNS with hard-coded IPs?

Only as a tightly controlled emergency measure for a very small dependency set. It creates stale routing, certificate, failover, and operational risks.

### How would you prove conntrack exhaustion?

Inspect `nf_conntrack_count`, `nf_conntrack_max`, kernel logs, conntrack statistics, and packet traces showing queries failing before they reach CoreDNS.

### How would you test TCP fallback?

Use `dig +tcp`, compare with UDP, and verify NetworkPolicy, firewall, and sidecar rules for TCP/53.

---

## Lab

### Objective

Reproduce and diagnose three DNS failure classes:

1. CoreDNS saturation or failure;
2. node-local resolver failure;
3. policy blocking TCP fallback.

### Environment

- local Kubernetes cluster or disposable cloud cluster;
- CoreDNS;
- optional NodeLocal DNSCache;
- optional Istio;
- a test workload with `dig`, `nslookup`, and `curl`.

### Exercise 1 — CoreDNS failure

- scale CoreDNS down or apply CPU pressure;
- observe latency and SERVFAIL behavior;
- compare existing and new client connections;
- restore CoreDNS and observe cache recovery.

### Exercise 2 — Node-specific failure

- break the node-local DNS listener on one node;
- schedule test pods on healthy and failing nodes;
- prove that direct CoreDNS pod queries still work;
- cordon and replace the failing node.

### Exercise 3 — TCP fallback

- block TCP/53 while allowing UDP/53;
- query a domain that produces a large response;
- compare `dig` and `dig +tcp`;
- repair policy and verify both paths.

### Success criteria

The student must identify the failed layer without restarting all workloads and produce:

- blast-radius statement;
- failure tree;
- packet-path evidence;
- immediate mitigation;
- permanent corrective actions;
- DNS SLO proposal.

---

## Mapping to experience

Prepare one production story using this structure:

- **Context:** cluster size, resolver architecture, mesh, and business impact.
- **Signal:** exact error, latency, and affected failure domain.
- **Investigation:** how you separated pod, node, service, CoreDNS, and upstream paths.
- **Mitigation:** how you preserved traffic and avoided restart amplification.
- **Root cause:** the precise failed layer.
- **Permanent fix:** architecture, rollout safety, monitoring, and client behavior.
- **Result:** reduced incident duration, lower DNS latency, or prevented recurrence.

A Principal-level story should show not only that you fixed DNS, but that you converted a hidden shared dependency into an observable, bounded, continuously tested platform capability.
