# Splunk → AWS S3 Glacier: Strategy & Design

**Related:** [Implementation Steps](./IMPLEMENTATION_STEPS.md) (phased execution, checklist)  
**Source:** Splunk + AWS Glacier Long-Term Archival Strategy  
**Selected storage class:** S3 Glacier Flexible Retrieval (Scenario C)  
**Target retention tier:** Post–Year 3 (Frozen / compliance archive)

---

## Objective

Extend the existing 3-year Splunk retention model with a **Cold → Frozen** path that archives frozen buckets to a dedicated S3 bucket, immediately transitions objects to **Glacier Flexible Retrieval**, encrypts data at rest (SSE-S3 AES-256), and supports auditable restore within one business day (Bulk retrieval: ~5–12 hours).

### Success criteria

| # | Criterion | How we verify |
|---|-----------|---------------|
| 1 | Buckets older than the configured cold retention are archived to `splunk-archive-logs` and removed from local cold storage | End-to-end dry-run + production pilot index |
| 2 | Objects land in Glacier Flexible Retrieval (0-day lifecycle) with SSE-S3 | AWS Console / CLI `head-object` + Storage Class |
| 3 | Archive failures are logged locally **and** alerted via SNS | Fault-injection test |
| 4 | A known frozen bucket can be thawed, rebuilt, and searched in Splunk | Restore drill |
| 5 | Steady-state storage cost tracks ~$0.0036/GB-month (Flexible Retrieval) for archived data | Cost Explorer tagged reports |

---

## Current architecture (as defined)

```
Year 0–1   Hot/Warm     → Fast disk (frequent search)
Years 2–3  Cold         → Searchable low-cost disk
Post-Year 3 Frozen      → coldToFrozenScript → S3 → Glacier Flexible Retrieval
```

Splunk invokes `coldToFrozenScript` when a bucket exceeds cold retention. The script must **reliably archive then dispose of** the local bucket directory; otherwise Splunk behavior and disk usage become undefined/unsafe.

---

## Gaps and risks in the source document (must resolve before build)

The strategy document is directionally correct but incomplete for production. Treat the items below as **blocking design decisions**, not optional polish.

### Script / Splunk contract gaps

| Gap | Risk | Required fix |
|-----|------|--------------|
| Script only uploads; does not remove/move the local bucket after success | Disk fills; duplicate archives; Splunk may re-invoke inconsistently | On **successful** upload only: delete the local bucket dir (or move to a quarantine path then delete after checksum verify) |
| No packaging (ZIP/TAR) despite “Best Practices” | High S3 PUT request cost; many tiny objects; slower restore | Archive the bucket directory to a single `.tar.gz` (or `.zip`) before upload |
| Object key = `basename(local_path)` only | Collisions across indexes / peers; restore ambiguity | Use hierarchical keys: `{index}/{peer_or_guid}/{bucket_id}/{archive_name}.tar.gz` |
| No checksum / integrity verification | Silent corruption or partial upload accepted as success | Compute SHA-256 (or use S3 multipart ETag carefully); verify after upload before local delete |
| No multipart upload | Large frozen buckets can fail or timeout | Use `boto3` managed transfer / multipart for files above a threshold (e.g. 100 MB) |
| Failure path alerts but leaves local state unclear | Ops may not know whether to retry manually | Define exit codes: `0` = archived & removed; non-zero = leave bucket in place for Splunk retry / manual intervention |
| Hard-coded region/account SNS ARN placeholders | Misconfiguration in prod | Externalize config (env vars or `/opt/splunk/etc/apps/.../local/archive.conf`) |
| Log path `/var/log/splunk/archive_errors.log` may not exist / not writable by Splunk user | Silent logging failure | Create path + ownership in deployment; also log to stdout/stderr for Splunk introspection where useful |

### AWS / security gaps

| Gap | Risk | Required fix |
|-----|------|--------------|
| IAM listed as `s3:PutObject` + `sns:Publish` only | Incomplete for multipart, listing, restore, inventory | Least-privilege policy covering Put/Get/AbortMultipart/ListBucket (prefix-scoped), `s3:RestoreObject`, SNS publish |
| “Glacier Vault Lock” called out for HIPAA-style retention | **Vault Lock applies to legacy Glacier vaults**, not S3 Glacier storage class | Use **S3 Object Lock** (Compliance or Governance mode) + retention period on the bucket, **or** a documented WORM alternative; do not implement Vault Lock against this S3 design |
| SSE-S3 with “AWS Managed Keys… restricted to Splunk IAM Role and Root” | SSE-S3 keys are not customer-restrictable like KMS CMKs | Confirm with Security: if key-access control / audit of decrypt is required, switch to **SSE-KMS** with a dedicated CMK and key policy; otherwise document SSE-S3 acceptance |
| Bucket public access / encryption defaults not specified | Accidental exposure | Block Public Access ON; default encryption SSE-S3 (or KMS); TLS-only bucket policy (`aws:SecureTransport`) |
| No cross-account / DR / versioning stance | Accidental overwrite / deletion | Decide: Object Lock **or** versioning + MFA delete / SCP deny-delete; document backup of archive metadata |

### Operations / restore gaps

| Gap | Risk | Required fix |
|-----|------|--------------|
| Restore steps omit restore duration days, retrieval tier selection, and expiration of temporary S3 copy | Cost spikes; restore fails mid-process | Formal thaw runbook with Bulk tier, restore days (e.g. 7), and cleanup |
| No inventory of what was archived (index, time range, S3 key, checksum) | Audits cannot find the right object | Maintain an archive manifest (local log + optional DynamoDB/S3 inventory + Splunk self-log) |
| `indexes.conf` snippet is incomplete / line-broken | Misconfiguration | Provide validated stanza with `coldToFrozenScript`, `coldToFrozenDir` decision, and per-index retention settings |
| No pilot / rollback plan | Fleet-wide disk or data-loss incident | Pilot on one non-critical index; feature flag via conf; ability to disable script and fall back to `coldToFrozenDir` |

### Cost model caveats (planning accuracy)

- Document rates (~$0.0036/GB-mo Flexible Retrieval, $360/mo for 100 TB) are **estimates** and region-specific; lock actual rates for the chosen region at design time.
- Bulk retrieval is **not free** at scale (per-GB retrieval + request fees). Document says “$0 retrieval using Bulk” — treat as **simplified**; finance should model expected audit retrieval volume.
- PUT/LIST/restore request charges and early-delete fees (if applicable to class/min storage duration) must be included.
- Packaging reduces request volume — include that assumption in the cost model.

---

## Design decisions (lock before implementation)

Complete this decision log in Phase 0; do not start coding until signed off.

| ID | Decision | Options | Recommendation | Owner |
|----|----------|---------|----------------|-------|
| D1 | Encryption | SSE-S3 vs SSE-KMS | Confirm with Security; default to **SSE-KMS** if key policy/audit required, else SSE-S3 per doc | Security + Splunk Platform |
| D2 | WORM / immutability | Object Lock Compliance vs Governance vs none | **Object Lock Governance** (or Compliance if legally mandated) with retention ≥ compliance window | Compliance |
| D3 | Archive format | `.tar.gz` vs `.zip` | **`.tar.gz`** (native on Linux indexers; preserves permissions/structure) | Platform |
| D4 | Object key scheme | Flat basename vs hierarchical | **Hierarchical**: `s3://splunk-archive-logs/{env}/{index}/{site}/{peer_name}/{bucket_id}.tar.gz` | Platform |
| D5 | Manifest store | File log only vs DynamoDB/S3 JSON sidecar | **Sidecar JSON next to archive** + append-only local/CSV or Splunk index for searchability | Platform |
| D6 | Failure retry | Rely on Splunk re-invoke vs external queue | Leave bucket in place on failure; alert; manual/automated retry job | Platform |
| D7 | Region | Single-region vs CRR | Single region matching Splunk footprint unless DR requires CRR | Cloud |
| D8 | Scope of indexes | All indexes vs allowlist | **Allowlist** pilot → expand | Platform |
| D9 | `coldToFrozenDir` vs script-only | Dir fallback vs script exclusive | Keep `coldToFrozenDir` as **emergency fallback** (disabled when script healthy) | Platform |
| D10 | Python runtime | System `/usr/bin/python3` vs Splunk’s Python | Prefer **system Python 3 + venv** with pinned `boto3`, or Splunk-supported method used elsewhere in the estate — pick one and standardize | Platform |

---

## Target technical design

### AWS resources

1. **S3 bucket** `splunk-archive-logs` (name may need account/env suffix for global uniqueness, e.g. `splunk-archive-logs-<account>-<region>`).
2. **Lifecycle rule:** Transition to `GLACIER` (Flexible Retrieval) after **0 days**.
3. **Default encryption:** SSE-S3 or SSE-KMS (per D1).
4. **Block Public Access:** all four settings ON.
5. **Bucket policy:** deny non-TLS; optional deny deletes if Object Lock not used.
6. **Object Lock** (if D2): enable at bucket creation (cannot retrofit easily — create correctly the first time).
7. **SNS topic** `SplunkArchiveAlerts` + subscriptions (email/PagerDuty/Slack).
8. **IAM role** on indexer instances (preferred over static keys):
   - `s3:PutObject`, `s3:GetObject`, `s3:AbortMultipartUpload`, `s3:ListBucket` (prefix-conditioned)
   - `s3:RestoreObject`, `s3:GetObjectVersion` (if versioning/Object Lock)
   - `sns:Publish` on the alerts topic
   - `kms:Encrypt/Decrypt/GenerateDataKey` if SSE-KMS
9. **CloudWatch / Cost Explorer tags:** `Application=Splunk`, `Purpose=FrozenArchive`, `Environment=...`

### Archive script (`archivetoglacier.py`) — required behavior

Deploy path (example): `/opt/splunk/bin/scripts/archivetoglacier.py`  
Invoked as: `coldToFrozenScript = /usr/bin/python3 /opt/splunk/bin/scripts/archivetoglacier.py`

**Processing pipeline:**

1. Receive Splunk bucket path (`sys.argv[1]`).
2. Validate path exists, is a directory, and looks like a Splunk bucket (presence of expected rawdata/journal structure — defensive check).
3. Resolve metadata: index name, bucket ID, peer/host, earliest/latest if discoverable.
4. Create compressed archive in a staging dir on a disk with enough free space (not inside the bucket path).
5. Upload to S3 with SSE, multipart as needed, object key per D4; optional metadata tags (`index`, `bucket_id`, `sha256`, `hostname`).
6. Verify upload (head-object + checksum compare / Content-Length).
7. Write manifest sidecar (local + optional S3 `.json` companion object in Standard or same lifecycle — prefer keeping small manifests in Standard via separate prefix **without** Glacier transition, if restore lookup speed matters).
8. On full success: securely delete local bucket directory; exit `0`.
9. On any failure: do **not** delete local bucket; log ERROR; SNS publish; exit non-zero.
10. Never upload secrets; never log full IAM credentials.

**Config (externalized):**

```text
BUCKET_NAME
SNS_TOPIC_ARN
AWS_REGION
LOG_FILE
STAGING_DIR
OBJECT_PREFIX / ENV
ENCRYPTION (AES256 | aws:kms)
KMS_KEY_ID (if applicable)
DRY_RUN (true/false)
```

### Splunk configuration

Per allowlisted index in `indexes.conf` (deploy via DS / cluster bundle):

```ini
[<index_name>]
# Existing hot/warm/cold sizing retained
coldToFrozenScript = /usr/bin/python3 /opt/splunk/bin/scripts/archivetoglacier.py
# Optional emergency fallback (document mutually exclusive ops procedure):
# coldToFrozenDir = /opt/splunk/frozen
```

Confirm retention math:

- Warm → cold transitions unchanged.
- Cold max age / size such that freeze occurs at **≥ 3 years** (or org policy), matching the strategy’s Post-Year 3 intent.
- Ensure frozen path does not conflict with homePath/coldPath capacity planning (staging space for tar).

Cluster considerations:

- Apply on **all indexers** that hold the index (or primary freezing peer behavior per Splunk clustering rules).
- Script and dependencies must be identical across peers (deploy as app or config management).

### Restore (thaw) design

Operational sequence (formalized from the doc):

1. Identify S3 object via manifest (index + time range + bucket_id).
2. `aws s3api restore-object` with **Bulk** tier and restore period (e.g. `Days=7`).
3. Wait until restore complete (poll `head-object` / Restore status) — expect **5–12 hours**.
4. Download archive to indexer `thaweddb` staging path (not into hot/cold paths).
5. Extract archive so Splunk sees a valid bucket directory under thaweddb.
6. `splunk rebuild <path_to_thawed_bucket>`
7. Verify searchable; document search constraints (thawed data availability).
8. After audit: remove thawed local data; allow S3 temporary restore copy to expire.

Provide a companion `thaw_from_glacier.py` (or runbook + AWS CLI) in Phase 4 — not required for Day-1 archive cutover, but required before declaring the program complete.

---

## Test plan

| Test | Type | Expected result |
|------|------|-----------------|
| IAM deny upload | Negative | Non-zero exit; bucket remains; SNS fires |
| SNS unavailable | Negative | Failure still logged locally; non-zero exit; bucket remains |
| Dry-run | Functional | Archive created in staging (or simulated); no S3 write; no delete |
| Small bucket E2E | Functional | Upload → Glacier class → local delete → manifest |
| Large bucket (>multipart threshold) | Functional | Successful multipart; verify integrity |
| Duplicate freeze retry | Safety | If local already gone, no-op/safe; if re-run against same dir prevented |
| Object key uniqueness | Functional | Two indexes with same bucket id suffix do not collide |
| Encryption | Security | `ServerSideEncryption` present on head-object |
| Restore Bulk | Ops | Searchable after rebuild within expected window |
| Cost sanity | Ops | Request counts drop when packaging enabled vs unpackaged baseline (lab) |

---

## Monitoring & alerting

| Signal | Where | Threshold / action |
|--------|-------|--------------------|
| Archive script failure | SNS + local log | Page on-call; investigate before disk fills |
| Indexer cold/thaw disk usage | Existing Splunk/infra monitors | Alert before staging cannot fit next freeze |
| S3 4xx/5xx on archive prefix | CloudWatch metrics | Investigate IAM/lifecycle/network |
| Unexpected retrieval costs | Budget alert | Confirm authorized thaw; check runaway restore jobs |
| Objects stuck in STANDARD (not transitioning) | S3 Inventory / lifecycle monitoring | Fix lifecycle rule / prefix filters |
| Missing manifests for archived objects | Periodic audit job | Repair process |

---

## Rollback & emergency procedures

1. **Disable freezing to Glacier:** remove `coldToFrozenScript` from affected indexes; deploy conf; verify no new script invocations.
2. **Fallback:** enable `coldToFrozenDir` to a local frozen volume with capacity alerts (temporary; costlier disk).
3. **Partial failure mid-rollout:** leave non-cutover indexes unchanged; fix script; re-enable per wave.
4. **Bad object in S3:** do not delete if compliance lock applies; upload corrected object under new key and update manifest; document supersession.
5. **Disk pressure during outage:** emergency expand cold volume; do not bypass checksum/delete safety to “make space” without incident commander approval.

---

## Artifacts to produce during build

| Artifact | Purpose |
|----------|---------|
| `scripts/archivetoglacier.py` | Cold-to-frozen archiver |
| `scripts/thaw_from_glacier.py` (or CLI runbook) | Controlled restore |
| `config/archive.env.example` | Externalized settings template |
| `apps/<splunk-app>/local/indexes.conf.example` | Splunk stanza examples |
| `iac/` (Terraform/CloudFormation) | Bucket, IAM, SNS, lifecycle, encryption |
| `docs/RESTORE_RUNBOOK.md` | Operator thaw procedure |
| `docs/DECISION_LOG.md` | D1–D10 outcomes |
| [Implementation Steps](./IMPLEMENTATION_STEPS.md) | Execution checklist |
| This document (`STRATEGY_AND_DESIGN.md`) | Strategy, design, and ops reference |

---

## Open questions for stakeholders (resolve in Phase 0)

1. Exact compliance retention **after** on-disk 3 years (e.g. +4 years in Glacier = 7 years total)?
2. Is SSE-S3 acceptable, or is SSE-KMS mandatory?
3. Is S3 Object Lock required? Compliance or Governance mode?
4. Which indexes are in-scope for v1 vs explicitly excluded (high-churn, non-regulated)?
5. Who is on-call for `SplunkArchiveAlerts`, and what is severity?
6. Is non-prod Splunk available for a full freeze/restore rehearsal?
7. Any requirement for cross-region replication or legal hold workflows?
8. Finance: expected number of audit retrievals per year for cost modeling beyond “Bulk ≈ $0”?

---

## Summary recommendation

Implement Scenario C (Glacier Flexible Retrieval) as specified, but **do not deploy the document’s script verbatim**. Harden it with packaging, hierarchical object keys, post-upload verification before local delete, externalized config, and a tested restore path. Create the S3 bucket only after immutability/encryption decisions, pilot on an allowlisted index, mandate a restore drill before fleet rollout, and operate with SNS + cost budgets as first-class controls.

Execute via [Implementation Steps](./IMPLEMENTATION_STEPS.md).
