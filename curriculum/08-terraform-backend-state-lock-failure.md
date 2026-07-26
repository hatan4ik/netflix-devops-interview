# Chapter 8 — Terraform Backend and State-Lock Failure

## Interview scenario

A production Terraform deployment is blocked. One pipeline reports that the remote state is locked, another engineer claims no apply is running, and the business is waiting on an urgent infrastructure change. The backend uses Amazon S3 for state and DynamoDB for locking.

The Staff/Principal-level task is not merely to remove a lock. It is to prove whether the lock is legitimate, preserve state integrity, restore safe delivery, and prevent recurrence.

---

## 1. What Terraform state actually represents

Terraform state is the control plane's memory of the relationship between configuration and real infrastructure.

It contains:

- resource addresses
- provider-specific IDs
- dependency relationships
- computed attributes
- outputs
- resource instances created with `count` or `for_each`
- sensitive values unless deliberately excluded or protected
- serial and lineage metadata

State is not just a cache. Corrupting or replacing it can cause Terraform to:

- recreate existing infrastructure
- delete the wrong resource
- lose ownership of live resources
- produce destructive plans
- expose secrets

The first principle is therefore:

> A blocked deployment is inconvenient; an incorrect state mutation can be catastrophic.

---

## 2. Remote backend mechanics

A common AWS backend looks like this:

```hcl
terraform {
  backend "s3" {
    bucket         = "prod-terraform-state"
    key            = "platform/network/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-locks"
  }
}
```

Conceptually:

```text
Terraform client
    |
    +--> DynamoDB conditional write: acquire lock
    |
    +--> S3 GET: read current state
    |
    +--> provider APIs: refresh, plan, apply
    |
    +--> S3 PUT: write new state
    |
    +--> DynamoDB delete: release lock
```

The lock prevents concurrent writers from applying against the same state snapshot.

Two applies against the same state without locking can both read serial N, make independent decisions, and then overwrite one another. The final state may describe only part of what actually changed.

---

## 3. What is inside a lock record

A lock record usually identifies:

- lock ID
- operation such as `OperationTypeApply`
- user or automation identity
- hostname or runner
- creation timestamp
- Terraform version
- state path
- informational metadata

The lock record is evidence. Do not delete it before capturing it.

Typical error:

```text
Error acquiring the state lock
ConditionalCheckFailedException: The conditional request failed
Lock Info:
  ID:        9f4d...
  Path:      prod-terraform-state/platform/network/terraform.tfstate
  Operation: OperationTypeApply
  Who:       terraform@runner-42
  Version:   1.x
  Created:   2026-07-26T20:15:11Z
```

---

## 4. Failure taxonomy

### 4.1 Legitimate active lock

An apply is still running, slow, paused, or waiting on a provider API.

Common examples:

- RDS modification
- EKS node-group update
- CloudFront distribution deployment
- IAM propagation delay
- a provider retry loop
- approval gate holding an active process

Removing this lock would permit a second writer and create a split-brain control-plane event.

### 4.2 Orphaned lock

The Terraform process died before releasing the lock.

Causes include:

- CI runner termination
- laptop shutdown
- network loss
- process kill
- OOM
- container eviction
- canceled job
- plugin crash

### 4.3 Wrong backend or wrong key

The operator may believe they are inspecting one environment while Terraform is using another.

Check:

- bucket
- key
- region
- workspace
- backend config supplied through `-backend-config`
- account and role identity
- environment-variable interpolation

### 4.4 IAM failure disguised as locking failure

The client may be unable to read, create, or delete the lock item.

Required permissions may include:

- `dynamodb:GetItem`
- `dynamodb:PutItem`
- `dynamodb:DeleteItem`
- `dynamodb:DescribeTable`
- `s3:GetObject`
- `s3:PutObject`
- `s3:ListBucket`
- KMS permissions when SSE-KMS is used

### 4.5 DynamoDB availability, throttling, or regional issue

A lock may be healthy while the lock service is unavailable or throttled.

### 4.6 S3 write failure after infrastructure mutation

This is one of the most dangerous cases.

Terraform may have changed real infrastructure but failed before persisting updated state. The lock might remain, and the old state may no longer match reality.

### 4.7 Manual state mutation

An engineer may have run:

- `terraform state rm`
- `terraform import`
- `terraform state mv`
- `terraform push`
- direct S3 replacement

The incident is then not simply a stale lock. It is a state-integrity investigation.

### 4.8 Multiple pipelines targeting one state

Separate repositories or jobs may unknowingly share the same backend key.

This creates chronic contention and an excessive blast radius.

---

## 5. Principal-level incident workflow

## Step 1 — Freeze competing writers

Pause or disable all pipelines targeting the same state key.

Do not allow retries to create noise while investigating.

Identify:

- repository
- workflow/job
- Terraform workspace
- backend bucket/key
- AWS account
- assumed role

## Step 2 — Capture the exact error and lock metadata

Record:

- lock ID
- path
- operation
- owner
- creation time
- Terraform version
- pipeline run ID

Preserve logs before runners expire.

## Step 3 — Confirm the actual backend

```bash
terraform init -reconfigure
terraform workspace show
aws sts get-caller-identity
```

Inspect `.terraform/terraform.tfstate` locally only to understand backend configuration; do not confuse it with infrastructure state.

## Step 4 — Determine whether the owner is alive

Check:

- CI system job status
- runner/pod/VM status
- process list
- CloudTrail activity from the lock owner
- provider-side operations still in progress
- Terraform logs

A job marked "canceled" is not enough proof. Its process may still be terminating or a cloud operation may still be underway.

## Step 5 — Inspect the lock item read-only

```bash
aws dynamodb get-item \
  --table-name terraform-state-locks \
  --key '{"LockID":{"S":"prod-terraform-state/platform/network/terraform.tfstate-md5"}}'
```

The exact key format depends on backend behavior and table contents. Enumerate carefully rather than guessing and deleting.

## Step 6 — Protect the current state

Before any force-unlock or repair:

```bash
terraform state pull > state-backup-$(date +%Y%m%d-%H%M%S).json
```

Also confirm S3 versioning and capture the current object version:

```bash
aws s3api list-object-versions \
  --bucket prod-terraform-state \
  --prefix platform/network/terraform.tfstate
```

Store the backup securely because state can contain secrets.

## Step 7 — Compare state with reality

Run a refresh-only plan when safe:

```bash
terraform plan -refresh-only
```

Do not jump directly to `apply`.

Look for:

- resources changed in the cloud but absent from state
- state entries whose real objects no longer exist
- replacements caused by immutable attributes
- unexpected account or region drift

## Step 8 — Remove the lock only with evidence

Preferred command:

```bash
terraform force-unlock <LOCK_ID>
```

Use it only after proving:

1. the original writer is dead
2. no provider operation is still being driven by that writer
3. all competing writers are frozen
4. state has been backed up
5. the correct backend and state key are confirmed

Do not make direct DynamoDB deletion the routine method. Direct deletion bypasses Terraform's own safety and audit context.

## Step 9 — Reconcile before resuming delivery

```bash
terraform plan -refresh-only -out=refresh.tfplan
terraform show refresh.tfplan
terraform plan -out=change.tfplan
terraform show change.tfplan
```

Require review for any destructive or replacement action.

## Step 10 — Resume one writer only

Re-enable a single controlled pipeline. Observe lock acquisition, apply, state write, and lock release.

---

## 6. Why `-lock=false` is usually the wrong answer

```bash
terraform apply -lock=false
```

This disables the protection preventing concurrent writers.

It may be acceptable only in exceptional read-oriented workflows where no write can occur and the risk is explicitly understood. It should not be used to bypass a production apply blockage.

A strong interview response says:

> I would rather delay the change than trade a visible lock incident for silent state corruption.

---

## 7. State serial, lineage, and overwrite protection

Terraform state includes lineage and serial metadata.

- **Lineage** identifies a state lineage.
- **Serial** increments as state changes.

These help prevent accidental replacement with unrelated or older state. They are not a substitute for locking, but they provide additional integrity checks.

Before restoring an S3 object version, compare:

- lineage
- serial
- resource inventory
- modification time
- real infrastructure state

Never restore an old object simply because it predates the incident. The older version may omit successfully created resources.

---

## 8. S3 backend hardening

Recommended controls:

- bucket versioning
- server-side encryption with KMS where appropriate
- public-access block
- restrictive bucket policy
- TLS-only access
- object access logging or CloudTrail data events
- separate state buckets or prefixes by trust boundary
- backup and restore testing
- lifecycle rules that retain enough historical versions
- no broad human write access

Example policy intent:

```text
CI role:
  read/write only its approved state prefix

Human operators:
  read by default
  break-glass write through audited role

Security administrators:
  KMS and bucket-policy administration

No principal:
  public access or unencrypted transport
```

---

## 9. DynamoDB lock-table hardening

- least-privilege IAM
- point-in-time recovery where organizational policy requires it
- CloudTrail monitoring
- alarms for throttling and access denial
- on-demand capacity or appropriately provisioned capacity
- one documented break-glass procedure
- clear ownership

Do not add TTL to lock records as a simplistic stale-lock solution. A legitimate apply can run longer than the TTL, after which a second writer could acquire the state while the first is still active.

Automatic lock expiry converts a fail-closed mechanism into a possible split-brain mechanism.

---

## 10. CI/CD design

A robust pipeline should include:

```text
format
  -> validate
  -> static/security checks
  -> init
  -> plan with lock timeout
  -> reviewed plan artifact
  -> single apply authority
  -> state-write verification
  -> post-apply checks
```

Useful command:

```bash
terraform plan -lock-timeout=10m
```

This tolerates legitimate short contention without disabling locking.

Design rules:

- one apply pipeline per state
- concurrency groups keyed by backend path
- canceling a workflow must terminate the Terraform process cleanly
- short-lived credentials
- no apply from developer laptops for production
- saved plan applied by the same trusted workflow
- provider lock file committed
- Terraform versions pinned
- manual approvals do not leave a process holding a lock unnecessarily

---

## 11. Reduce blast radius through state decomposition

A single state containing an entire company creates:

- large lock contention
- slow plans
- broad credentials
- large failure domains
- risky ownership boundaries

Split state by stable operational boundary, for example:

```text
organization/bootstrap
network/shared
cluster/prod-us-east-1
cluster/prod-us-west-2
data/prod
service/catalog
```

Do not fragment so aggressively that every output becomes a brittle remote-state dependency.

Good boundaries align with:

- team ownership
- lifecycle
- privileges
- change frequency
- blast radius

---

## 12. Recovery scenarios

### Scenario A — Pipeline died before making changes

1. verify owner is dead
2. back up state
3. force-unlock
4. run refresh-only plan
5. run normal plan
6. resume one writer

### Scenario B — Infrastructure changed but state write failed

1. freeze writers
2. collect logs and provider operation IDs
3. back up all state versions
4. inspect real resources
5. use refresh/import/state repair deliberately
6. peer review every state mutation
7. produce a clean plan before apply

### Scenario C — State object was overwritten incorrectly

1. freeze writers
2. preserve current and historical versions
3. compare serial and lineage
4. reconcile each disputed resource with reality
5. restore or reconstruct state only after review

### Scenario D — Lock owner is still active

Do not unlock. Let the operation finish, or terminate it through a controlled process and then verify it is truly dead.

---

## 13. Observability and alerting

Monitor:

- lock acquisition latency
- lock contention count
- lock age
- failed state writes
- S3 `AccessDenied`
- DynamoDB conditional-check failures
- DynamoDB throttles
- apply duration
- abandoned CI jobs
- frequency of force-unlock usage

Alerting should distinguish:

- expected short contention
- stale locks
- backend availability failures
- unauthorized access
- state write failures after provider mutations

Every force-unlock should create an auditable event with owner, reason, evidence, backup location, approver, and follow-up ticket.

---

## 14. Security considerations

State commonly contains sensitive data, including values returned by providers.

Controls:

- encrypt at rest and in transit
- restrict access by state prefix
- avoid outputting secrets where possible
- use secret managers rather than treating state as one
- redact state from CI logs
- never attach raw production state to an ordinary ticket
- rotate credentials exposed through state history
- audit state access

The lock table itself may reveal environment names, usernames, paths, and infrastructure structure. Restrict it accordingly.

---

## 15. Debugging command set

```bash
# Identity
aws sts get-caller-identity

# Terraform version and providers
terraform version
terraform providers

# Backend initialization
terraform init -reconfigure

# Workspace
terraform workspace show
terraform workspace list

# State backup
terraform state pull > state-backup.json

# Read-only reconciliation
terraform plan -refresh-only

# Force unlock after evidence collection
terraform force-unlock <LOCK_ID>

# S3 object history
aws s3api list-object-versions \
  --bucket <bucket> \
  --prefix <key>

# Lock table inspection
aws dynamodb scan \
  --table-name <lock-table> \
  --max-items 20

# CloudTrail investigation
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=PutItem
```

Use DynamoDB `scan` sparingly in large tables; a targeted `get-item` is preferable once the exact key is known.

---

## 16. Anti-patterns

- deleting the lock immediately
- running `apply -lock=false`
- assuming a CI cancellation killed the process
- restoring an older state without checking real resources
- editing JSON manually as the first response
- allowing many repositories to write the same state
- storing all environments in one state
- granting broad human access to the state bucket
- adding TTL-based automatic lock deletion
- applying an unreviewed plan immediately after unlock

---

## 17. Principal-level interview answer

> I would first stop all writers targeting that backend key and capture the lock metadata. Then I would verify the exact S3 bucket, key, workspace, AWS account, and role, because many apparent stale locks are actually backend-selection mistakes. I would determine whether the lock owner is still alive by checking the CI job, runner, process, provider-side operations, and CloudTrail activity. Before changing anything, I would pull and securely preserve the current state and confirm S3 object version history. If the writer is definitively dead, I would use `terraform force-unlock` with the recorded lock ID rather than bypass locking. I would then run a refresh-only plan to reconcile state with reality, review any drift or replacement, and resume exactly one controlled writer. Longer term, I would enforce one apply authority per state, CI concurrency keyed by backend path, S3 versioning, least-privilege access, lock and state-write observability, and state decomposition by ownership and blast radius. I would never solve a production lock incident with `-lock=false`, because that replaces a visible delivery delay with the possibility of silent state corruption.

---

## 18. Follow-up interview questions

### Why not just delete the DynamoDB row?

Because the row may represent an active writer. Direct deletion bypasses Terraform's safety workflow and can allow concurrent state mutation.

### What if the CI job says canceled?

Prove the Terraform process and provider operations are no longer active. Cancellation status alone is insufficient.

### What if S3 state write failed after resource creation?

Freeze writers, preserve state versions, inspect real infrastructure, and deliberately reconcile through refresh, import, or reviewed state operations.

### Would you use TTL for stale locks?

No. Long legitimate applies can outlive the TTL, allowing concurrent writers.

### How do you reduce lock contention?

Use one apply authority, concurrency controls, sensible `-lock-timeout`, shorter plans, and state decomposition along ownership and lifecycle boundaries.

### Is S3 versioning enough?

No. It helps recovery but does not prevent concurrent writers or guarantee the newest historical version is correct.

---

## 19. Whiteboard model

```text
                    +--------------------+
                    | CI concurrency key |
                    | bucket/key/workspace|
                    +----------+---------+
                               |
                               v
+-------------+      +-------------------+      +----------------+
| Terraform   |----->| DynamoDB lock     |----->| exclusive      |
| runner      |      | conditional write |      | writer         |
+------+------+      +-------------------+      +--------+-------+
       |                                                |
       | S3 GET                                         | provider APIs
       v                                                v
+-------------------+                          +-------------------+
| versioned S3      |<-------------------------| infrastructure    |
| state object      |       state commit       | control planes    |
+-------------------+                          +-------------------+
```

The key invariant is:

> At most one authorized writer may mutate a state lineage at a time, and every infrastructure mutation must eventually be reconciled into a durable, versioned state record.

---

## 20. Practical lab

Build a disposable environment with:

- S3 versioned bucket
- DynamoDB lock table
- two isolated Terraform runners
- one small AWS resource

Exercises:

1. acquire a lock with runner A
2. attempt a plan from runner B
3. inspect the lock record
4. terminate runner A abruptly
5. back up state
6. prove the owner is dead
7. force-unlock
8. run refresh-only plan
9. simulate a provider mutation followed by state-write failure
10. reconcile the resource using import or refresh
11. inspect S3 object versions
12. add CI concurrency controls

Success criteria:

- no concurrent applies
- no state loss
- reproducible recovery
- auditable unlock procedure
- clean final plan

---

## 21. Staff/Principal signals

A strong candidate:

- treats state as critical control-plane data
- distinguishes active, orphaned, and inaccessible locks
- verifies backend identity before acting
- protects evidence and state versions
- reconciles real infrastructure after failures
- rejects unsafe shortcuts
- designs organizational controls, not only commands
- reduces future blast radius

A weak candidate says only:

```bash
terraform force-unlock <id>
```

The command is easy. The engineering judgment is proving that it is safe.
