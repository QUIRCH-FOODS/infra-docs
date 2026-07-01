# WORM (Immutable Storage) — qsqlbackups / qfsql-daily

**Status:** Policy applied **Unlocked** on 2026-06-30; re-verified still **Unlocked** on 2026-07-01 (no changes). Next step: validate one nightly backup cycle, then **lock** to meet the security requirement. Until locked, the policy is **not** compliant — locking is the step that satisfies the requirement.

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
- "Append blobs" chosen because SQL `BACKUP ... TO URL` / SQL MI backups write **append blobs** in chunks; "None" would fail the backup job mid-write.

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
