# Chapter 13 — Sudden NAT Gateway Cost Increase

## Interview scenario

AWS networking cost rises sharply even though no VPC, subnet, route table, or NAT Gateway was intentionally changed. The increase appears under NAT Gateway processing and related data-transfer line items. Applications still work, infrastructure health is green, and the traffic increase may be concentrated in one account, Region, Availability Zone, cluster, node group, workload, or destination.

You are the Staff/Principal SRE leading cost containment, traffic forensics, service protection, and durable redesign.

---

## 1. What the interviewer is testing

A strong answer must show that you can:

- treat cloud cost as an observable production signal rather than an accounting surprise;
- distinguish NAT hourly cost from per-byte processing and related transfer charges;
- move from a billing line item to a specific NAT Gateway, ENI, workload, destination, and request class;
- reason about route tables, DNS, VPC endpoints, cross-AZ paths, retries, image pulls, telemetry, and exfiltration;
- contain spend without breaking egress, allowlists, updates, or critical dependencies;
- choose between NAT, gateway endpoints, interface endpoints, direct internet egress, proxies, and centralized egress based on measured traffic and security requirements;
- add cost attribution, anomaly detection, and route-path validation so the class of incident does not recur.

The weak answer is: “Add an S3 endpoint” or “delete the NAT Gateway.”

The Principal answer is:

> A NAT bill increase means more bytes, more gateways, or a different charged path. I first scope the exact account, Region, usage type, and time window; correlate the cost delta with NAT CloudWatch byte metrics; identify the NAT ENI; then use VPC Flow Logs and workload inventory to find top sources and destinations. I contain the dominant flow safely, verify whether a private path regressed to public egress, and redesign the route so cost, availability, and security are explicit.

---

## 2. Build the cost model before investigating

For a conventional public NAT Gateway, the cost model usually contains at least:

```text
NAT cost
  = NAT Gateway hourly charges
  + NAT data-processing charges
  + applicable data-transfer charges
  + optional Transit Gateway / firewall / endpoint charges in the path
```

A useful incident approximation is:

```text
incremental NAT processing cost
  ≈ incremental processed GB × regional NAT processing rate
```

If a cross-AZ path is involved, the full incremental cost may include additional data-transfer components on top of NAT processing.

### 2.1 Do not hard-code one universal rate

Rates vary by Region and service path and can change. Use the AWS pricing page, Cost Explorer, or the Cost and Usage Report for the exact environment and time period.

### 2.2 Bytes can be charged even when the request fails

Retries, partial downloads, failed handshakes, and timeout loops can all process bytes. A failed user transaction can therefore cost more than a successful one.

### 2.3 “No infrastructure change” is not “no path change”

The path can change because of:

- DNS behavior;
- VPC endpoint removal or private DNS changes;
- route-table association drift;
- application destination changes;
- container base-image changes;
- retry-policy changes;
- node churn;
- new telemetry or backup behavior;
- a dependency moving from private to public addressing;
- security compromise or unexpected data transfer.

---

## 3. Understand the traffic path

A common zonal design is:

```text
private workload in AZ-a
        |
        v
private subnet route table
0.0.0.0/0 -> NAT Gateway in AZ-a
        |
        v
public subnet route table
0.0.0.0/0 -> Internet Gateway
        |
        v
public destination
```

A problematic cross-AZ design may be:

```text
private workload in AZ-b
        |
        v
NAT Gateway in AZ-a
        |
        v
Internet Gateway
```

The second path can add cross-AZ data-transfer cost and couples AZ-b internet egress to a NAT resource in AZ-a.

AWS also offers other NAT and egress architectures. The same forensic method still applies: establish the actual route, resource, byte count, and billing dimensions before redesigning.

### 3.1 NAT is not the route-policy owner

The NAT Gateway translates traffic. Subnet route tables, endpoint routes, DNS answers, and application destinations determine which traffic reaches it.

A NAT spike is therefore often a routing, naming, or application-behavior incident—not a NAT appliance incident.

---

## 4. First-response strategy: STABILIZE

### Step 1 — Bound the financial and service impact

Record:

- payer and linked account;
- Region;
- VPC and NAT Gateway ID;
- Availability Zone;
- billing usage type and operation;
- start time and slope of increase;
- hourly versus data-processing component;
- related inter-AZ, internet egress, Transit Gateway, firewall, or PrivateLink changes;
- affected service and customer impact.

### Step 2 — Freeze changes that destroy evidence

Pause or carefully control:

- node-group rollout;
- image refresh;
- telemetry-agent rollout;
- endpoint or route-table updates;
- DNS resolver changes;
- retry-policy changes;
- backup or data-migration jobs;
- broad Pod restarts.

Do not delete the NAT Gateway before mapping the traffic. That can remove service connectivity and erase the easiest correlation point.

### Step 3 — Correlate cost with bytes

Compare the billing delta with NAT Gateway CloudWatch metrics for the same time window.

If cost rises and byte metrics rise proportionally, focus on traffic volume and path.

If hourly cost rises without a matching byte increase, look for newly created NAT Gateways, duplicated environments, or idle gateways.

If transfer charges rise more than NAT processing, investigate cross-AZ, inter-Region, Transit Gateway, or internet egress dimensions.

### Step 4 — Protect service while containing spend

Potential containment actions include:

- stop a runaway batch or backup job;
- disable a noncritical exporter;
- reduce retry amplification;
- restore a known-good private endpoint or route-table association;
- shift traffic away from an affected workload cohort;
- block a confirmed unauthorized destination;
- rate-limit package or artifact downloads;
- restore node/image caching behavior;
- disable a faulty feature flag.

Use change control. A cost incident can be a security incident or an availability incident in disguise.

---

## 5. Billing forensics

### 5.1 Cost Explorer

Group the affected period by:

- service;
- linked account;
- Region;
- usage type;
- operation;
- Availability Zone when available;
- cost-allocation tags where present.

Look for NAT-related usage types, but also inspect adjacent categories such as data transfer, Transit Gateway processing, Network Firewall, PrivateLink, and internet egress.

### 5.2 Cost and Usage Report query

A CUR-backed Athena query can identify the dominant NAT dimensions. Adjust database, table, and column names to the organization’s CUR schema.

```sql
SELECT
    line_item_usage_account_id                  AS account_id,
    product_region                              AS region,
    line_item_usage_type                        AS usage_type,
    line_item_operation                         AS operation,
    date_trunc('hour', line_item_usage_start_date) AS hour,
    SUM(line_item_usage_amount)                 AS usage_amount,
    SUM(line_item_unblended_cost)               AS unblended_cost
FROM cur_database.cur_table
WHERE line_item_usage_start_date >= TIMESTAMP '2026-07-25 00:00:00'
  AND line_item_usage_start_date <  TIMESTAMP '2026-07-27 00:00:00'
  AND lower(line_item_usage_type) LIKE '%natgateway%'
GROUP BY 1,2,3,4,5
ORDER BY hour, unblended_cost DESC;
```

Then compare the incident period with a representative baseline:

```sql
WITH hourly AS (
  SELECT
      date_trunc('hour', line_item_usage_start_date) AS hour,
      line_item_usage_account_id AS account_id,
      product_region AS region,
      line_item_usage_type AS usage_type,
      SUM(line_item_unblended_cost) AS cost
  FROM cur_database.cur_table
  WHERE line_item_usage_start_date >= TIMESTAMP '2026-07-01 00:00:00'
    AND lower(line_item_usage_type) LIKE '%natgateway%'
  GROUP BY 1,2,3,4
)
SELECT *
FROM hourly
ORDER BY cost DESC
LIMIT 200;
```

### 5.3 A disciplined cost bridge

Build a bridge such as:

```text
baseline NAT processing bytes
+ new image-pull bytes
+ retry-amplified API bytes
+ cross-AZ rerouted bytes
+ telemetry-export bytes
= observed processed-byte increase
```

The incident is not complete until the traffic explanation approximately reconciles with the billed and metered increase.

---

## 6. NAT Gateway CloudWatch metrics

Important metrics include:

- `BytesInFromSource` — bytes received from clients in the VPC;
- `BytesOutToDestination` — bytes sent toward destinations;
- `BytesInFromDestination` — bytes received from destinations;
- `BytesOutToSource` — bytes returned to VPC clients;
- `PacketsInFromSource` and related packet counters;
- `ActiveConnectionCount`;
- `ConnectionAttemptCount` and established-connection metrics;
- `ErrorPortAllocation`;
- `PacketsDropCount`;
- `IdleTimeoutCount`.

Interpret directions carefully. For example:

- large downloads often increase bytes from destination and bytes out to source;
- uploads often increase bytes in from source and bytes out to destination;
- connection storms may increase attempts and port-allocation pressure without proportionally large payload bytes.

List NAT metrics:

```bash
aws cloudwatch list-metrics \
  --namespace AWS/NATGateway \
  --dimensions Name=NatGatewayId,Value=nat-0123456789abcdef0
```

Fetch a one-minute series:

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/NATGateway \
  --metric-name BytesOutToDestination \
  --dimensions Name=NatGatewayId,Value=nat-0123456789abcdef0 \
  --statistics Sum \
  --period 60 \
  --start-time 2026-07-26T00:00:00Z \
  --end-time 2026-07-26T06:00:00Z
```

### 6.1 Packet-drop percentage

A useful derived signal is:

```text
packet drop percentage
  = PacketsDropCount
    / (PacketsInFromSource + PacketsInFromDestination)
    × 100
```

### 6.2 Cost and reliability can share one root cause

Examples:

- retry storms increase bytes and connection attempts;
- short-lived connections increase port pressure and TLS overhead;
- port-allocation errors cause failures and retries;
- idle connection churn raises attempts and application latency;
- packet loss increases retransmission and processed bytes.

Do not investigate cost separately from error rate, latency, and saturation.

---

## 7. Map the NAT Gateway to its network interfaces and route users

Describe NAT Gateways:

```bash
aws ec2 describe-nat-gateways \
  --filter Name=state,Values=available \
  --query 'NatGateways[].{
    NatGatewayId:NatGatewayId,
    VpcId:VpcId,
    SubnetId:SubnetId,
    Addresses:NatGatewayAddresses,
    Tags:Tags
  }'
```

Inspect routes that target the NAT Gateway:

```bash
aws ec2 describe-route-tables \
  --filters Name=route.nat-gateway-id,Values=nat-0123456789abcdef0
```

Identify:

- associated private subnets;
- their Availability Zones;
- whether each subnet routes to an AZ-local egress path;
- whether a route was replaced or propagated unexpectedly;
- whether gateway-endpoint prefix-list routes are present;
- whether a centralized egress VPC or Transit Gateway is involved.

Inspect subnet and NAT placement:

```bash
aws ec2 describe-subnets \
  --subnet-ids subnet-aaaa subnet-bbbb \
  --query 'Subnets[].{SubnetId:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock}'
```

---

## 8. VPC Flow Logs: identify top talkers

Enable or use existing Flow Logs at the appropriate scope:

- NAT Gateway network interface;
- private subnets;
- workload ENIs;
- VPC;
- Transit Gateway when relevant.

Capture accepted and rejected traffic where policy permits. Include fields useful for translated or intermediary traffic, such as:

- `interface-id`;
- `srcaddr`, `dstaddr`;
- `pkt-srcaddr`, `pkt-dstaddr`;
- source and destination ports;
- protocol;
- packets and bytes;
- action;
- start and end time;
- flow direction;
- traffic path;
- VPC, subnet, instance, and account identifiers when available.

### 8.1 Athena top-source query

Adjust names to the deployed Flow Log schema.

```sql
SELECT
    coalesce(pkt_srcaddr, srcaddr) AS source_ip,
    SUM(bytes) AS total_bytes,
    SUM(packets) AS total_packets,
    COUNT(*) AS flow_records
FROM vpc_flow_logs
WHERE interface_id = 'eni-natgateway'
  AND action = 'ACCEPT'
  AND from_unixtime(start) >= TIMESTAMP '2026-07-26 00:00:00'
  AND from_unixtime(start) <  TIMESTAMP '2026-07-26 06:00:00'
GROUP BY 1
ORDER BY total_bytes DESC
LIMIT 100;
```

### 8.2 Top destinations and ports

```sql
SELECT
    coalesce(pkt_dstaddr, dstaddr) AS destination_ip,
    dstport,
    protocol,
    SUM(bytes) AS total_bytes,
    SUM(packets) AS total_packets
FROM vpc_flow_logs
WHERE interface_id = 'eni-natgateway'
  AND action = 'ACCEPT'
  AND from_unixtime(start) BETWEEN
      TIMESTAMP '2026-07-26 00:00:00'
      AND TIMESTAMP '2026-07-26 06:00:00'
GROUP BY 1,2,3
ORDER BY total_bytes DESC
LIMIT 200;
```

### 8.3 Compare incident and baseline

```sql
SELECT
    coalesce(pkt_dstaddr, dstaddr) AS destination_ip,
    SUM(CASE
          WHEN from_unixtime(start) >= TIMESTAMP '2026-07-26 00:00:00'
           AND from_unixtime(start) <  TIMESTAMP '2026-07-26 06:00:00'
          THEN bytes ELSE 0 END) AS incident_bytes,
    SUM(CASE
          WHEN from_unixtime(start) >= TIMESTAMP '2026-07-19 00:00:00'
           AND from_unixtime(start) <  TIMESTAMP '2026-07-19 06:00:00'
          THEN bytes ELSE 0 END) AS baseline_bytes
FROM vpc_flow_logs
WHERE interface_id = 'eni-natgateway'
GROUP BY 1
ORDER BY incident_bytes - baseline_bytes DESC
LIMIT 100;
```

### 8.4 Limitations

Flow Logs are essential but not a full packet trace or billing ledger.

They may not directly provide:

- DNS name;
- HTTP path;
- application identity;
- exact retried user request;
- TLS SNI in ordinary flow records;
- business owner;
- perfect one-to-one reconciliation with every billing line item.

Join them with DNS query logs, proxy logs, application telemetry, CloudTrail, Kubernetes inventory, and cost data.

---

## 9. Map source IPs to workloads

### 9.1 EC2 and ENIs

```bash
aws ec2 describe-network-interfaces \
  --filters Name=addresses.private-ip-address,Values=10.20.30.40 \
  --query 'NetworkInterfaces[].{
    ENI:NetworkInterfaceId,
    Description:Description,
    InstanceId:Attachment.InstanceId,
    SubnetId:SubnetId,
    PrivateIps:PrivateIpAddresses[*].PrivateIpAddress,
    Groups:Groups[*].GroupId,
    Tags:TagSet
  }'
```

### 9.2 EKS Pods

For current state:

```bash
kubectl get pods -A -o wide | grep '10.20.30.40'
```

Or generate an inventory:

```bash
kubectl get pods -A -o json | jq -r '
  .items[] |
  [.metadata.namespace,
   .metadata.name,
   .spec.nodeName,
   .status.podIP,
   (.metadata.labels.app // "-")] |
  @tsv'
```

Historical attribution requires preserved inventory because Pods disappear and IPs are reused. Store periodic mappings of:

- Pod UID;
- namespace and workload owner;
- node;
- Pod IP;
- ENI and subnet where available;
- image digest;
- deployment revision;
- team and cost-center metadata.

### 9.3 Other source types

Map sources to:

- Lambda ENIs;
- ECS tasks;
- EC2 instances;
- databases with unexpected egress;
- build systems;
- self-hosted CI runners;
- NAT behind Transit Gateway;
- Network Firewall or inspection VPCs;
- third-party appliances.

---

## 10. Map destination IPs to services and domains

An IP is often insufficient because CDNs and cloud services share or rotate addresses.

Use:

- Route 53 Resolver query logs;
- application access logs;
- Envoy or egress-proxy logs;
- DNS cache logs;
- TLS SNI where explicitly collected and permitted;
- service endpoint inventories;
- AWS-managed prefix lists;
- VPC endpoint and route-table configuration;
- threat-intelligence and ASN ownership for security triage.

### 10.1 Route 53 Resolver query-log correlation

Correlate:

```text
source workload IP
+ query time
+ queried domain
+ returned address
+ NAT Flow Log destination
```

This can distinguish:

- an AWS service public endpoint;
- a third-party API;
- image registry;
- package mirror;
- telemetry collector;
- content origin;
- suspicious destination.

### 10.2 Beware DNS caching and connection reuse

The DNS query may occur minutes before the flow. Preserve a window wide enough to account for TTL and application caching.

---

## 11. Common root causes

### 11.1 Gateway endpoint route regression

S3 and DynamoDB gateway endpoints can keep supported traffic off NAT paths.

A spike can occur when:

- the endpoint was deleted;
- a new private-subnet route table was not associated;
- a route table was replaced during Terraform changes;
- traffic changed Region;
- policy or application naming caused requests to use a different endpoint;
- a new subnet or node group missed the endpoint route.

Verify:

```bash
aws ec2 describe-vpc-endpoints \
  --filters Name=vpc-id,Values=vpc-0123456789abcdef0

aws ec2 describe-route-tables \
  --route-table-ids rtb-0123456789abcdef0
```

### 11.2 Interface endpoint or private-DNS regression

For services using interface endpoints, traffic may start using public service addresses when:

- private DNS was disabled;
- endpoint DNS names were not used as expected;
- endpoint association or resolver rules changed;
- endpoint security groups blocked the required clients;
- a new VPC or subnet was omitted;
- application configuration forced a public endpoint.

Do not assume an application automatically and safely falls back. Prove the DNS answer and route from the affected workload.

### 11.3 ECR, image-layer, and package-download storms

Typical triggers:

- node replacement or autoscaling;
- image cache loss;
- mutable tags causing repeated pulls;
- `imagePullPolicy: Always` combined with frequent restarts;
- new, much larger images;
- package installation at container startup;
- CI runners pulling dependencies through production NAT;
- missing ECR API, ECR Docker, S3, or other required private paths.

Inspect:

```bash
kubectl get events -A --sort-by=.lastTimestamp | grep -i -E 'pull|image'
kubectl get pods -A -o json | jq -r '.items[].spec.containers[].image' | sort | uniq -c
```

### 11.4 Retry amplification after dependency slowdown

A dependency slowdown can increase NAT bytes even if business QPS is flat.

Track:

```text
user requests
versus
application attempts
versus
proxy attempts
versus
NAT connections and bytes
```

One retry owner, bounded attempts, backoff, jitter, and deadline propagation are cost controls as well as reliability controls.

### 11.5 Telemetry-export change

Examples:

- traces sampled at 100% instead of 1%;
- log payloads include large request/response bodies;
- metrics remote-write duplicates data;
- exporter switched from a private collector to a public SaaS endpoint;
- collector retries indefinitely;
- compression was disabled;
- a new agent runs on every node or Pod.

Compare exporter bytes per request and per host before and after the incident.

### 11.6 Backup, replication, reindex, or data migration

A one-time operational job can dominate NAT traffic:

- database export to a public endpoint;
- object copy through an application host;
- cross-Region replication through the wrong path;
- search reindex downloading source data;
- media transformation pulling source assets;
- disaster-recovery test;
- developer data sync.

Jobs need explicit network and dollar budgets, owners, and stop conditions.

### 11.7 Cross-AZ NAT routing

In zonal NAT designs, private subnets should generally use an AZ-local NAT path when the architecture calls for high availability and avoidance of cross-AZ transfer.

Check:

- subnet AZ;
- NAT subnet AZ;
- route-table association;
- Transit Gateway appliance mode and symmetry where relevant;
- centralized inspection path;
- failover route that remained active after an incident.

### 11.8 Public endpoint selected instead of private service path

Causes include:

- endpoint override in application configuration;
- DNS search-path or resolver changes;
- Private Hosted Zone disassociation;
- split-horizon DNS error;
- service discovery returning public names;
- an SDK Region mismatch;
- direct public hostname embedded in code.

### 11.9 Cache miss or content-origin amplification

For streaming systems, a cache-policy or key-format change can send large media or metadata requests from private compute through NAT to a public origin.

Investigate:

- hit ratio by region and request class;
- object size;
- range-request behavior;
- key-version changes;
- cache eviction;
- CDN/origin routing;
- retries and partial downloads;
- prefetch or speculative fetch volume.

### 11.10 Security compromise or data exfiltration

Red flags:

- new destination countries or ASNs;
- unusual ports;
- sustained encrypted upload;
- one workload with no legitimate egress need;
- traffic outside deployment windows;
- new credentials or role sessions;
- crypto-mining pool or command-and-control patterns;
- large archive creation followed by egress.

Actions:

1. engage security incident response;
2. preserve Flow Logs, CloudTrail, DNS logs, process evidence, and container images;
3. isolate the workload with controlled network policy or security groups;
4. revoke compromised credentials;
5. block confirmed malicious destinations;
6. assess data exposure;
7. avoid destructive remediation before evidence capture unless active harm requires it.

### 11.11 Port allocation and connection churn

A NAT Gateway must allocate source ports for connections. High concurrency to a small destination tuple, very short-lived connections, or poor connection reuse can cause `ErrorPortAllocation`.

Symptoms:

- connection timeouts;
- retries;
- increased `ConnectionAttemptCount`;
- `ErrorPortAllocation` greater than zero;
- high `PacketsDropCount`;
- cost rising from repeated failed attempts.

Improve connection reuse, distribute destinations where architecturally valid, reduce unnecessary retries, and follow AWS guidance for the selected NAT architecture.

### 11.12 Idle timeout mismatch

Long-idle connections may be closed and recreated. If clients handle resets poorly, this can create reconnect loops and duplicate work.

Correlate `IdleTimeoutCount` with:

- application reconnects;
- keepalive configuration;
- connection-pool churn;
- TLS handshakes;
- retries and request duplication.

---

## 12. Cost attribution for Kubernetes and platform teams

A production platform should expose NAT cost drivers by ownership boundary.

Recommended dimensions:

- account;
- Region and AZ;
- VPC and NAT Gateway;
- cluster;
- namespace;
- workload owner;
- source subnet and node group;
- destination category;
- protocol and port;
- bytes and connections;
- deployment revision;
- business request volume.

### 12.1 Unit economics

Useful metrics include:

```text
NAT GB per 1,000 playback starts
NAT GB per 1,000 API requests
NAT GB per node replacement
NAT GB per image deployment
NAT GB per GB of customer content delivered
NAT cost per environment per day
```

These metrics distinguish legitimate growth from path inefficiency.

### 12.2 Ownership metadata

Require tags and inventory fields such as:

- `Owner`;
- `Service`;
- `Environment`;
- `CostCenter`;
- `DataClassification`;
- `EgressPolicy`;
- `Cluster`;
- `Expiration` for temporary infrastructure.

Tags alone do not attribute shared NAT bytes. Join tagging with Flow Logs and workload inventory.

---

## 13. Safe architecture decisions

### 13.1 Gateway endpoints

For supported S3 and DynamoDB traffic, gateway endpoints can provide private connectivity without an internet gateway or NAT device for that service path.

Validate:

- Region;
- route-table associations;
- endpoint policy;
- bucket/table policy interactions;
- DNS and application endpoint behavior;
- failover and disaster-recovery requirements.

### 13.2 Interface endpoints / AWS PrivateLink

Interface endpoints can privately expose supported AWS or partner services through ENIs.

Trade-offs:

- hourly endpoint cost;
- per-GB processing cost;
- one or more ENIs per AZ;
- security groups;
- private DNS;
- endpoint quotas;
- centralized versus distributed ownership;
- possible cross-AZ traffic if clients use nonlocal endpoint ENIs.

Do not replace NAT with dozens of interface endpoints without comparing the actual hourly and data-processing economics.

### 13.3 Per-AZ NAT architecture

For zonal NAT Gateway deployments, one NAT Gateway per active AZ and AZ-local routing can improve failure isolation and avoid unnecessary cross-AZ traversal.

Trade-offs:

- more hourly gateway charges;
- less cross-AZ dependency and transfer;
- more route tables and controls;
- clearer AZ-level attribution.

The correct choice depends on traffic volume, availability targets, and the current AWS NAT offering in the Region.

### 13.4 Centralized egress VPC

A centralized architecture can improve:

- policy consistency;
- firewall inspection;
- static allowlisted addresses;
- audit and ownership.

It can also add:

- Transit Gateway processing;
- cross-AZ transfer;
- larger blast radius;
- routing complexity;
- asymmetric path risk;
- scaling and failure-domain concerns.

Compare the full path cost, not only NAT Gateway count.

### 13.5 Egress proxy

An authenticated egress proxy can provide:

- domain-aware policy;
- destination attribution;
- connection reuse;
- request logs;
- explicit allowlists;
- centralized TLS policy where appropriate.

Risks include:

- a new shared failure domain;
- privacy and sensitive-log concerns;
- protocol limitations;
- certificate-management complexity;
- proxy cost and scaling.

### 13.6 Direct public IP or Internet Gateway path

Some workloads may not require NAT if they can safely use public addressing and strict security controls. This is a security and architecture decision—not merely a cost optimization.

Consider:

- inbound exposure;
- static egress IP requirements;
- compliance;
- workload identity;
- firewalling;
- IPv4 scarcity;
- operational ownership.

### 13.7 IPv6

IPv6 can avoid some IPv4 NAT requirements, but it changes routing, security, DNS, dependency compatibility, and monitoring. An egress-only internet gateway, DNS64/NAT64 requirements, and dual-stack behavior must be designed deliberately.

Do not describe IPv6 as a free drop-in NAT replacement.

---

## 14. Preventive controls

### 14.1 Cost anomaly detection

Alert on:

- absolute NAT cost;
- percentage change from baseline;
- processed GB per hour;
- NAT GB per business transaction;
- new NAT Gateway count;
- new destination category;
- one source exceeding its normal share;
- cross-AZ path activation;
- unexplained transfer cost paired with NAT growth.

Use seasonal baselines. Weekend, release, and major-content traffic may differ.

### 14.2 CloudWatch alarms

Create alarms for:

- unexpected byte rate;
- `ErrorPortAllocation > 0`;
- packet-drop percentage;
- idle timeout anomalies;
- connection-attempt anomalies;
- sudden active-connection growth.

### 14.3 Route and endpoint policy as code

CI checks can verify:

- every private subnet has the intended egress route;
- zonal subnets use the intended NAT resource;
- required gateway endpoint routes exist;
- interface endpoints have private DNS and expected security groups;
- no unauthorized `0.0.0.0/0` route was introduced;
- route-table associations match subnet ownership;
- endpoint policies and resource policies are compatible;
- temporary exceptions have expiration metadata.

### 14.4 Synthetic path tests

From representative subnets and workloads, test:

- DNS answer is private where expected;
- route reaches the endpoint, not NAT;
- security groups and endpoint policies permit access;
- AWS service calls succeed;
- public internet access remains available only where intended;
- failover preserves the desired path.

### 14.5 Egress budgets

Define budgets by workload:

```text
maximum external GB/hour
maximum destination QPS
maximum retry multiplier
maximum image bytes per rollout
maximum telemetry bytes per request
maximum backup transfer rate
```

Enforce with rate limits, concurrency controls, deployment gates, and alerts where practical.

### 14.6 Historical network inventory

Retain enough history to answer:

- which Pod owned IP `10.20.30.40` at 02:17 UTC;
- which image and revision it ran;
- which node and subnet hosted it;
- which team owned it;
- which DNS names it resolved;
- which external destinations it contacted.

Without historical attribution, Flow Logs may reveal the bytes but not the responsible code.

---

## 15. Hands-on investigation lab

### Lab objective

Create and diagnose multiple NAT-cost spikes while preserving application availability.

### Environment

- two or three Availability Zones;
- private subnets;
- NAT Gateway egress;
- EKS or EC2 workloads;
- VPC Flow Logs to S3;
- Athena table;
- CloudWatch dashboards;
- optional S3 gateway endpoint and one interface endpoint;
- Route 53 Resolver query logging;
- synthetic traffic generator.

### Experiment 1 — Remove an S3 gateway-endpoint route association

1. Establish baseline S3 traffic.
2. Remove the endpoint association from one private-subnet route table.
3. Verify the affected subnet uses NAT.
4. Correlate NAT bytes, Flow Logs, route table, and workload.
5. Restore the route and validate recovery.

### Experiment 2 — Cross-AZ NAT route

1. Route an AZ-b private subnet through an AZ-a NAT Gateway.
2. Generate controlled traffic.
3. identify the path and related cost dimensions.
4. Restore AZ-local routing.

### Experiment 3 — Image-pull storm

1. Use a large image and replace nodes.
2. Measure image-pull bytes and Pod events.
3. Add the intended private service paths or improve image strategy.
4. repeat and compare.

### Experiment 4 — Retry storm

1. Introduce latency or errors in a public dependency.
2. Enable retries at application and proxy layers.
3. compare user requests, attempts, NAT connections, and bytes.
4. assign one retry owner and retest.

### Experiment 5 — Telemetry misconfiguration

1. Increase trace sampling and disable compression in a test exporter.
2. identify the top source and destination.
3. restore bounded sampling and compression.

### Experiment 6 — Suspicious egress

1. Generate a controlled unexpected upload from one workload.
2. detect it through anomaly, Flow Logs, and DNS correlation.
3. isolate the workload and preserve evidence.

### Exit criteria

The learner must produce:

- a billing-scope statement;
- CloudWatch correlation;
- top-source and top-destination Flow Log queries;
- source-IP-to-workload mapping;
- a path diagram;
- a cost bridge that explains most of the increase;
- a safe containment action;
- a durable architecture fix;
- monitoring and ownership controls.

---

## 16. Principal-level 90-second answer

> I would treat the NAT increase as a production traffic-forensics incident. First I would scope the exact linked account, Region, usage type, gateway count, and time window in Cost Explorer or CUR. Then I would correlate the delta with the NAT Gateway’s CloudWatch byte, connection, port-allocation, packet-drop, and idle-timeout metrics. That tells me whether I am paying for more gateways, more bytes, or a failing connection pattern.
>
> Next I would identify the NAT Gateway’s ENI and every subnet route that uses it. I would query VPC Flow Logs for the top packet source, destination, port, and byte deltas, then map source IPs to EC2, EKS Pods, Lambda, ECS, or Transit Gateway traffic. I would correlate destination IPs with Resolver query logs and proxy/application logs. My leading hypotheses are a private endpoint or route regression, cross-AZ routing, image or package pulls after node churn, telemetry export, retries after a dependency slowdown, a bulk-data job, or unauthorized exfiltration.
>
> I would contain the largest confirmed flow without deleting shared egress or causing an origin outage. The durable fix could be a gateway endpoint, correctly designed interface endpoint, AZ-local NAT path, controlled egress proxy, retry budget, image-distribution improvement, or workload policy. I would close by adding NAT GB per business transaction, top-talker attribution, route tests, cost anomalies, and security alerts so the next path change is detected within minutes rather than on the monthly bill.

---

## 17. Likely follow-up questions

### Why can NAT cost rise with no Terraform change?

Because DNS, routes selected by new subnets, application destinations, retries, node churn, telemetry, or data jobs can change the traffic path or byte volume without changing the NAT resource itself.

### Why not create an interface endpoint for every AWS service?

Interface endpoints have hourly, per-AZ, and processing costs. The correct design depends on traffic volume, endpoint count, availability, security, and cross-AZ behavior.

### How do you prove an S3 endpoint regression?

Show that the affected subnet route table lacks the expected endpoint prefix-list route, the workload resolves and reaches the service through the NAT path, Flow Logs identify the traffic, and restoring the endpoint route removes the NAT bytes.

### Why can retries increase cost dramatically?

Every attempt can create connections and transfer request or response bytes. Multiple retry layers multiply attempts even when customer request volume is flat.

### How do you attribute NAT traffic to a Kubernetes Pod?

Use `pkt-srcaddr` or the relevant Flow Log source field, map the IP to current or historical Pod inventory, then join it with node, ENI, namespace, owner, image digest, DNS logs, and deployment revision.

### What if Flow Logs show only destination IPs from a CDN?

Correlate Resolver query logs, SNI or egress-proxy logs where permitted, application request logs, and timestamped DNS answers. IP ownership alone may not identify the business dependency.

### Is one NAT Gateway per AZ always cheapest?

No. It can reduce cross-AZ traffic and improve failure isolation but adds hourly gateway cost. Compare measured hourly, per-GB, and transfer economics with availability requirements.

### What security signal would make you escalate immediately?

A new workload or destination with sustained unexplained outbound transfer, unusual port or geography, unexpected credential activity, or evidence of archive-and-upload behavior.

### Which single metric is insufficient?

Total NAT bytes. It proves volume, not source, destination, business cause, route correctness, security legitimacy, or the complete cost path.

---

## 18. Review checklist

Before declaring the incident resolved, confirm:

- [ ] exact billing account, Region, usage type, and time window are identified;
- [ ] hourly versus processing versus transfer components are separated;
- [ ] NAT CloudWatch bytes reconcile directionally with the cost increase;
- [ ] NAT Gateway and associated route tables are mapped;
- [ ] top source workloads are identified;
- [ ] top destinations and domains are identified;
- [ ] cross-AZ and centralized-egress paths are understood;
- [ ] endpoint and private-DNS behavior are verified from affected workloads;
- [ ] retry and connection amplification are measured;
- [ ] image, package, telemetry, backup, and cache behaviors are checked;
- [ ] security reviewed unexplained egress;
- [ ] containment did not break critical egress;
- [ ] the cost bridge explains most of the observed delta;
- [ ] the durable route or application fix is deployed;
- [ ] anomaly detection and top-talker attribution exist;
- [ ] historical IP-to-workload inventory is retained;
- [ ] ownership and egress budgets are documented.

---

## 19. References

- AWS VPC: Pricing for NAT Gateways — https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-pricing.html
- AWS VPC: NAT Gateway Metrics and Dimensions — https://docs.aws.amazon.com/vpc/latest/userguide/metrics-dimensions-nat-gateway.html
- AWS VPC: View NAT Gateway CloudWatch Metrics — https://docs.aws.amazon.com/vpc/latest/userguide/viewing-metrics.html
- AWS VPC: Troubleshoot NAT Gateways — https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-troubleshooting.html
- AWS VPC: Work with NAT Gateways — https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-working-with.html
- AWS VPC: Flow Log Record Fields — https://docs.aws.amazon.com/vpc/latest/userguide/flow-log-records.html
- AWS VPC: Query Flow Logs with Athena — https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs-athena.html
- AWS VPC: Gateway Endpoints — https://docs.aws.amazon.com/vpc/latest/privatelink/gateway-endpoints.html
- Amazon EKS Best Practices: Networking Cost Optimization — https://docs.aws.amazon.com/eks/latest/best-practices/cost-opt-networking.html

---

## Core principle

> NAT spend is traffic evidence. The engineering objective is not merely to make the bill smaller; it is to prove which workload sent which bytes over which route, contain unsafe behavior, and make the cheapest secure and resilient path the default.
