# Splunk → AWS S3 Glacier: Implementation Steps

**Related:** [Strategy & Design](./STRATEGY_AND_DESIGN.md) (objectives, gaps, decisions, design, testing, ops)  
**Source strategy:** Splunk + AWS Glacier Long-Term Archival Strategy  
**Selected storage class:** S3 Glacier Flexible Retrieval (Scenario C)

Do **not** enable `coldToFrozenScript` in production before Phases 1–2 are complete and a non-prod (or carefully controlled) freeze has succeeded. Do **not** declare the program done before Phase 4 restore drill passes. Lock decisions D1–D10 in [Strategy & Design](./STRATEGY_AND_DESIGN.md) before Phase 1 build.

---

## Delivery sequence

```text
Phase 0  Decisions & capacity
    ↓
Phase 1  AWS bucket/IAM/SNS  ─── (Object Lock must be decided here)
    ↓
Phase 2  Script + tests + packaging
    ↓
Phase 3  Non-prod rehearsal → Prod pilot index
    ↓
Phase 4  Restore drill + runbook sign-off
    ↓
Phase 5  Wave rollout
    ↓
Phase 6  Hardening, budgets, quarterly drills
```

---

## Phase 0 — Prerequisites & decisions (gate)

**Goals:** Align Security/Compliance/Finance; prevent irreversible bucket misconfiguration.

**Work:**

1. Confirm compliance retention (total years in Glacier beyond the 3 on-disk years).
2. Sign decision log D1–D10 (see [Strategy & Design](./STRATEGY_AND_DESIGN.md#design-decisions-lock-before-implementation)).
3. Choose AWS account, region, naming, tagging standard.
4. Confirm indexer OS, Python 3 version, outbound network to S3/SNS/STS (and VPC endpoints if private).
5. Disk capacity check: cold volume + staging headroom for largest expected frozen bucket × compression factor.
6. Identify pilot index (low business risk, known retention).
7. Confirm whether Object Lock is required **before** bucket creation.

**Exit criteria:** Written approvals for encryption, immutability, pilot index, and cost envelope.

---

## Phase 1 — AWS foundation

**Goals:** Create a correctly configured, empty archival landing zone.

**Work:**

1. Create SNS topic + test subscription; confirm alert delivery.
2. Create IAM role/instance profile + least-privilege policies; attach to pilot indexer(s).
3. Create S3 bucket with:
   - Block Public Access
   - Default encryption
   - Object Lock (if approved)
   - Lifecycle → Glacier Flexible Retrieval @ 0 days (exclude `manifests/` prefix if using Standard manifests)
   - TLS-only bucket policy
   - Versioning if required by Object Lock / ops
4. Optional: VPC gateway endpoint for S3; interface endpoint considerations for SNS.
5. Apply Cost Allocation tags.
6. Smoke-test: manual encrypted upload from pilot indexer role; confirm storage class transition; confirm deny without TLS / without role.

**Exit criteria:** Manual upload from indexer role succeeds; object shows Glacier Flexible Retrieval after lifecycle; SNS test message received; unauthorized principals cannot write.

---

## Phase 2 — Archive script engineering

**Goals:** Production-ready `archivetoglacier.py` replacing the document’s happy-path skeleton.

**Work:**

1. Implement packaging, hierarchical keys, checksums, multipart upload, config externalization, exit-code contract, structured logging.
2. Implement SNS alerting on failure (retain nested try/except so SNS failure still logs).
3. Add `--dry-run` mode (package + log intended S3 key; no upload/delete).
4. Unit tests where practical (path validation, key builder, failure paths mocked).
5. Package dependencies (`boto3`/`botocore` pinned) via venv or system packages managed by CM.
6. Create log directory + logrotate for `archive_errors.log` / success audit log.
7. Deploy script to pilot indexer via the same pipeline used for Splunk apps/CM.

**Exit criteria:** Code review passed; dry-run against a sample bucket directory succeeds; fault injection (bad bucket name / denied IAM) produces log + SNS and leaves data intact.

---

## Phase 3 — Splunk pilot integration

**Goals:** Prove Cold→Frozen on one index without fleet risk.

**Work:**

1. Deploy script + IAM to pilot indexer(s).
2. Configure pilot index `coldToFrozenScript` in a test/app local conf; push bundle.
3. Force a freeze in a controlled way (lab bucket or shortened retention in non-prod **only**), or wait for natural freeze in lower environment first.
4. Preferred path: **non-prod / lab Splunk** full rehearsal before prod pilot.
5. Validate:
   - Local bucket removed only after successful archive
   - Object in S3 with correct key, encryption, tags, checksum
   - Lifecycle → Glacier
   - Manifest written
   - Splunk continues healthy (disk, indexing, searches)
6. Monitor for 1–2 freeze cycles on the pilot index.

**Rollback:** Remove/comment `coldToFrozenScript`; optionally enable `coldToFrozenDir`; restart/refresh as required; stop script deploys.

**Exit criteria:** ≥1 successful automated freeze archive in the target environment class (non-prod then prod pilot); zero unalerted failures.

---

## Phase 4 — Restore drill (mandatory before broad rollout)

**Goals:** Prove audit recoverability within the Scenario C business-day window.

**Work:**

1. Document and execute thaw runbook on the pilot archive.
2. Time each stage (restore request → available → download → rebuild → searchable).
3. Capture actual retrieval cost for the sample.
4. Fix gaps (permissions, paths, rebuild issues).
5. Store runbook in ops wiki / repo (`docs/RESTORE_RUNBOOK.md`).

**Exit criteria:** Independent operator (not the author) completes restore using the runbook; data searchable; timing acceptable to Compliance.

---

## Phase 5 — Fleet rollout

**Goals:** Expand allowlist carefully.

**Work:**

1. Roll IAM + script to all indexers via CM / deployment app.
2. Enable indexes in waves (e.g. 10% → 50% → 100% of allowlisted indexes).
3. Watch disk free space, SNS alerts, S3 PUT error rates, Cost Explorer daily.
4. Disable any legacy local frozen dirs once confident (keep documented emergency procedure).
5. Update capacity plans: reclaim cold disk growth expectations post–Year 3.

**Exit criteria:** All in-scope indexes freezing successfully for an agreed observation window; alert noise tuned; cost within envelope.

---

## Phase 6 — Hardening & steady state

**Goals:** Compliance and cost hygiene.

**Work:**

1. Enable/confirm Object Lock retention or SCP deny-delete controls.
2. S3 Inventory + Athena or manifest search for audit “what do we have?”
3. Cost Explorer budgets + anomaly alerts (especially retrieval spikes).
4. Quarterly restore drill.
5. Review Deep Archive only for true “read almost never” cohorts (optional later optimization — out of initial scope).
6. Patch dependencies (`boto3`) on a cadence; retest upload/restore after major changes.

**Exit criteria:** Runbooks owned; budget alarms live; quarterly drill scheduled.

---

## Work breakdown checklist

### AWS / Cloud

- [ ] Decision log signed (D1–D10)
- [ ] SNS topic + subscriptions
- [ ] IAM role/policies + instance profile on indexers
- [ ] S3 bucket (encryption, BPA, lifecycle, Object Lock if required, TLS deny)
- [ ] Tags + Cost Explorer/budget
- [ ] (Optional) VPC endpoints

### Script / packaging

- [ ] `archivetoglacier.py` with full success/failure contract
- [ ] External config
- [ ] Staging + `tar.gz` packaging
- [ ] Hierarchical keys + metadata/tags
- [ ] Checksum verify before local delete
- [ ] Multipart upload
- [ ] Local logging + logrotate
- [ ] Dry-run mode
- [ ] Dependency install method documented

### Splunk

- [ ] Deployment artifact (app or CM state) for script + conf
- [ ] Pilot `indexes.conf` stanza
- [ ] Retention settings validated against 3-year policy
- [ ] Cluster bundle validation / rolling apply procedure
- [ ] Disk staging capacity confirmed

### Ops / compliance

- [ ] Restore runbook + companion tooling
- [ ] Archive manifest/search procedure for audits
- [ ] On-call alert routing for SNS
- [ ] Rollback procedure
- [ ] Quarterly drill calendar
