# WORM (Immutable Storage) — qsqlbackups / qfsql-daily

Knowledge record:
- Topic: WORM policy compatibility incident for SQL MI backup target
- Owning repo: https://github.com/QUIRCH-FOODS/infra-docs
- Canonical guide: https://github.com/QUIRCH-FOODS/infra-docs/blob/main/docs/worm-qfsql-daily.md
- Canonical script/config: https://github.com/QUIRCH-FOODS/infra-docs/blob/main/docs/worm-qfsql-daily.md
- Last verified: 2026-07-21
- Usage intent: Use this before enabling immutability on active SQL MI backup containers.

Exception classification:
- Allowed local content under the Knowledge Index Standard as incident history, architecture decision context, and environment-specific operational risk.

> **🛑 DO NOT LOCK. Incident 2026-07-01: this policy broke the SQL MI backup job.** The "validate one cycle then lock" plan below is **superseded** — validation *failed*. Time-based WORM immutability is architecturally incompatible with SQL MI `BACKUP ... TO URL`, and the `allowProtectedAppendWrites` exemption does **not** fix it (backups are block blobs, not append blobs). See **[Incident / root cause](#incident--root-cause-2026-07-01)** below. Fix requires the admin account (this analysis was done via read-only QAzureReadOnlyApp). Remove/adjust the policy — do not lock it, or production backups become permanently impossible.

**Status:** Policy applied **Unlocked** on 2026-06-30. **2026-07-01: confirmed BREAKING the nightly `qfsql-daily` backup job** — every backup fails with `BlobImmutableDueToPolicy`. Policy is still Unlocked (reversible). **Do not lock.**

## Incident / root cause (2026-07-01)

**Symptom:** SQL job writing backups to `qfsql-daily` via SAS started failing right after WORM was enabled (2026-06-30). Nothing lands in the container; each failed run leaves an orphaned 0-byte `.bak`.

**Root cause:** SQL Managed Instance `BACKUP ... TO URL` writes **block blobs** (`PutBlob` → `PutBlock` → `PutBlockList`), **not append blobs**. Under time-based retention, the initial empty `PutBlob` commits a blob that is *immediately* immutable for the retention period; the subsequent `PutBlock` that streams the data is then rejected. `allowProtectedAppendWrites=true` only exempts **append-blob** `AppendBlock` ops — it has no effect on block-blob backups. **SQL MI backups and time-based WORM on the same blobs cannot coexist.**

**Evidence** — Log Analytics workspace `Security` (customerId `24f0fdd8-08a9-4088-aa0c-a9851539472b`), table `StorageBlobLogs`, blob-service diagnostic setting `Blob00` (audit category → workspace `Security`). Per-blob failing sequence (SAS, UA `SQLBLOBACCESS`, caller `192.168.126.151`):

| Step | Operation | Result |
|------|-----------|--------|
| 1 | `PutBlob` (empty 0-byte block blob) | 201 Success |
| 2 | `AcquireBlobLease` | 201 Success |
| 3 | **`PutBlock`** (stage data block) | **409 `BlobImmutableDueToPolicy`** ← fails |
| 4 | `DeleteBlob` (SQL cleanup) | **409 `BlobImmutableDueToPolicy`** ← orphan can't be removed |

Timeline (daily `PutBlock` on `qfsql-daily`): Jun 25 = 25,857 ok / 0 fail; Jun 29 = 59,547 / 0; Jun 30 = 59,573 / 0; **Jul 01 = 0 committed / 11 `BlobImmutableDueToPolicy`.** DBs hit on the 04:00 run: QFERP, QFERPStaging, QFERPHistory, CBBStaging, QFMigration, QFR, QFWeb, AIDocVaultDB.

**Options (all need admin / write identity):**
1. **Remove WORM from `qfsql-daily`** now (reversible while Unlocked) to restore backups; meet the retention requirement via #2.
2. **Separate write from retain:** keep the backup container mutable; copy sealed `.bak` files into a *separate* immutable container (or use Lifecycle Management). Immutability only works on blobs never modified after commit.
3. Version-level immutability — complex and still fragile with SQL's multi-block write; not recommended for an active backup target.

Recommendation: **#1 now**, then design retention via #2.

---

## (Superseded) original validate-then-lock plan

The plan below assumed backups would keep working under WORM. They do not — see the incident above. Kept for reference only.

## Resource
- **Storage account:** `qsqlbackups` (StorageV2, Standard_RAGRS, eastus2)
- **Subscription / RG:** QuirchFoodsSubscription (`2ea66c5f-74a0-4844-801e-e17a7cd5c9b3`) / `ProductionGroup`
- **Container:** `qfsql-daily`

## Policy (verified via QAzureReadOnlyApp, 2026-06-30)
| Setting | Value | Meaning |
|---------|-------|---------|
| Policy type | Time-based retention | Immutable for N days from each blob's creation timestamp |
| `immutabilityPeriodSinceCreationInDays` | **6** | Each blob un-deletable/un-modifiable for 6 days from its own creation; older blobs become deletable |
| `state` | **Unlocked** | Still adjustable/removable; NOT yet compliant — must be locked |
| `allowProtectedAppendWrites` | **true** | "Append blobs" — permits SQL backup-to-URL append operations under WORM |
| `allowProtectedAppendWritesAll` | false | "Block and append blobs" NOT selected |
| version-level immutability | disabled | Container-scoped, not per-version |
| etag (for --if-match) | `"8ded6da11c87b53"` | Pass when locking / increasing period |

## Key facts about this WORM config
- Immutability is **per-blob from creation time**, not a rolling container window. A blob is protected for its first 6 days, then becomes deletable. WORM does **not** auto-delete — it only blocks deletion while young. Old-backup cleanup must be done by the backup job or a Lifecycle Management rule.
- **Increasing** the period is always allowed (locked or unlocked); locked allows up to 5 increases. **Decreasing** is only possible while unlocked. Locking is **irreversible** — cannot shorten, remove, or delete the container/account until all blobs' retention expires.
- ~~"Append blobs" chosen because SQL `BACKUP ... TO URL` / SQL MI backups write **append blobs** in chunks~~ **← WRONG, this was the flawed assumption.** SQL MI backups write **block blobs** (`PutBlob`/`PutBlock`/`PutBlockList`), so `allowProtectedAppendWrites` never applied — see [Incident / root cause](#incident--root-cause-2026-07-01).

## Before locking (do not skip — locking is irreversible)
Locking cannot be undone, so validate first while the policy is still Unlocked:
1. Let one nightly `qfsql-daily` backup run under the policy.
2. Confirm the SQL backup job **succeeds** and a new blob lands (no WORM append/overwrite write failures).
3. Only then lock (write op — needs an admin identity; QAzureReadOnlyApp is read-only and cannot lock):
```bash
az storage container immutability-policy lock \
  --account-name qsqlbackups --container-name qfsql-daily \
  --resource-group ProductionGroup --if-match '"8ded6da11c87b53"' \
  --subscription 2ea66c5f-74a0-4844-801e-e17a7cd5c9b3 --auth-mode login
```
(etag changes after any modification — re-read with `immutability-policy show` if lock rejects the if-match.)
