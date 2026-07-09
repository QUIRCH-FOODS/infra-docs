# Security Assessment — AccountsPayableDocumentReaderV2

**Resource:** Standard Logic App `AccountsPayableDocumentReaderV2`, resource group `Dev_RG`, QuirchFoodsSubscription, East US 2
**Date:** 2026-07-09
**Context:** Cloned from Consumption Logic App `AccountsPayableDocumentReader`, reconfigured to reach the `qfsql` SQL Managed Instance via VNet integration (`QSegmentedVnet` / `QSegmentedSubnet2`) instead of the on-premises data gateway on VM `QSQL`.

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

5. **Unresolved RBAC grants at `Dev_RG` scope** (inherited by this app): three service principals with **Contributor** resolve only as bare GUIDs (`e2a3a434-...`, `b2c00ee4-...`, `2ab69b3f-...`) — worth confirming what they are, since Contributor here means they can modify/delete this app and its settings (including the secret in #1). Also flagging `SQL-MI-InstanceAdmin` holding **SQL Managed Instance Contributor** on `Dev_RG` — an unusual role/scope pairing unrelated to this app's resource group.

## Priority order
Start with #3 (network restriction) and #5 (RBAC audit) — highest leverage relative to effort. #1/#2 are inherent to the Azure Files limitation and lower urgency unless compliance requires it.
