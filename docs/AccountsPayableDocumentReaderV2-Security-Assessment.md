# Security Assessment — AccountsPayableDocumentReaderV2

**Resource:** Standard Logic App `AccountsPayableDocumentReaderV2`, resource group `Dev_RG`, QuirchFoodsSubscription, East US 2
**Date:** 2026-07-09 (findings live-verified 2026-07-10 via QAzureReadOnlyApp)
**Context:** Cloned from Consumption Logic App `AccountsPayableDocumentReader`, reconfigured to reach the `qfsql` SQL Managed Instance via VNet integration (`QSegmentedVnet` / `QSegmentedSubnet2`) instead of the on-premises data gateway on VM `QSQL`.

> **2026-07-10 verification note:** The app is now **live and healthy**, not disabled. App Insights (`AppInsights-AccountsPayableDocumentReaderV2`) shows a full end-to-end run 15:05–16:03 UTC on 2026-07-10 — trigger 51×, SQL sproc lookups 16×, file writes 14×, SQL inserts 14×, email moves 14× — **0 failures, 0 exceptions**. SQL dependency 60 calls / 0 failed confirms the `qfsql` built-in-connector VNet path works under load. Findings #2/#3/#4 below re-confirmed against live config; #5 principals resolved.

## What's solid

| Control | Status |
|---|---|
| HTTPS enforced | ✅ `httpsOnly: true` |
| Minimum TLS version | ✅ TLS 1.2 (site and storage account) |
| FTP | ✅ `FtpsOnly` — plain FTP disabled |
| SCM/FTP Basic Auth publishing | ✅ Disabled on both |
| Host storage auth | ✅ Managed identity (`AzureWebJobsStorage__credential=managedidentity`), not a connection string |
| Storage blob public access | ✅ Disabled (`allowBlobPublicAccess: false`) |
| UAMI role scope | ✅ Scoped to just this one storage account, not subscription-wide |

## Findings to act on

1. **Plaintext storage account key in app settings.** `WEBSITE_CONTENTAZUREFILECONNECTIONSTRING` contains the full account key in cleartext — unavoidable today since Azure Files content-share connections don't support managed identity. Mitigations: rotate the key periodically, and/or move it to a Key Vault reference (`@Microsoft.KeyVault(...)`) instead of storing it inline.

2. **`allowSharedKeyAccess: true` on the storage account.** The account key (exposed per #1) remains a valid, fully-privileged credential even though the app's primary path uses managed identity. If nothing legitimately needs key-based access, disable shared key access entirely to neutralize #1 rather than just mitigate it.

3. **No network restriction on storage account or site** (`defaultAction: Allow` / `publicNetworkAccess: Enabled`). Given VNet integration is already in place for reaching `qfsql`, and the trigger is email-polling (not an inbound webhook), there's no obvious need for public reachability. Consider a private endpoint on the storage account or at minimum "allow trusted Microsoft services."

4. **No IP/access restrictions on the workflow trigger.** Access control is unset — only RBAC governs who can hit management endpoints. Low risk given the trigger type, but no network-layer backstop exists.

5. **Broad RBAC grants at `Dev_RG` scope** (inherited by this app): three service principals hold **Contributor**. As of 2026-07-10 these resolved (via Graph) to first-party Quirch app-deployment identities — **not** unknown/external principals:
   - `e2a3a434-...` → `quirchfoods-QScanner-...` (created 2025-03-03)
   - `b2c00ee4-...` → `quirchfoods-QFERP-...` (created 2025-04-10)
   - `2ab69b3f-...` → `quirchfoods-QFWEB-...` (created 2025-12-04)

   Origin is benign, but the exposure stands: each can modify/delete this app and its settings (including the secret in #1) because Contributor is granted at the whole resource-group scope. Recommend right-sizing each to the specific resources it actually deploys to. Also flagging `SQL-MI-InstanceAdmin` (group) holding **SQL Managed Instance Contributor** on `Dev_RG` — an unusual role/scope pairing unrelated to this app.

6. **No monitoring alerts on this app.** Confirmed 2026-07-10: of 494 metric alerts in the subscription (plus scheduled-query and activity-log alerts), **none** are scoped to `AccountsPayableDocumentReaderV2` or its App Insights. Now that the workflow runs unattended in production, a silent failure (expired O365 connection, broken VNet path, host down) would go undetected — the app only emails on errors its *own* logic catches. Add at minimum a workflow-run-failure alert and a trigger-heartbeat/"no runs" alert.

## Live config confirmations (2026-07-10)
- `allowSharedKeyAccess: true` on `accountpayableslogicapp` (finding #2) — confirmed.
- `publicNetworkAccess: Enabled` + `defaultAction: Allow` on both `accountpayableslogicapp` and `qfilestoragestandard`; site `publicNetworkAccess: Enabled` (finding #3) — confirmed.
- Site + SCM `ipSecurityRestrictions` = `Allow / Any` (finding #4) — confirmed. Note `clientCertMode: Required` is set, a partial inbound backstop but not a substitute for network scoping.

## Priority order
Start with #3 (network restriction), #6 (add alerts), and #5 (right-size Contributor scopes) — highest leverage relative to effort. #1/#2 are inherent to the Azure Files limitation and lower urgency unless compliance requires it.
