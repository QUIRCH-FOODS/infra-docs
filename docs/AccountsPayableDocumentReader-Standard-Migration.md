# AccountsPayableDocumentReader — Consumption → Standard Migration

**Date:** 2026-07-09
**Goal:** Move `AccountsPayableDocumentReader` off the on-premises data gateway (VM `QSQL`) and onto direct VNet connectivity to the `qfsql` SQL Managed Instance, using Azure Logic Apps' "Clone to Standard (Preview)" feature.

## Source and destination
- **Source (Consumption):** `AccountsPayableDocumentReader`, resource group `Dev_RG`, QuirchFoodsSubscription, EastUS2.
- **Destination (Standard):** `AccountsPayableDocumentReaderV2`, same resource group/subscription/region. Workflow name: `AccountsPayableDocumentReader_WFv2`.
- Hosting: shared App Service Plan `ASP-ProductionGroup-9c92` (WorkflowStandard tier, WS1) — also hosts `LLMIA`, `LogicDocuments`, `QF-EDI`, `qmaster-logicapps`. Standard Logic Apps bill per plan, not per app, so reusing this plan was free.
- Identity: user-assigned managed identity `AccountsPayableDocumentReaderV2-uami`, used for host storage (`AzureWebJobsStorage__credential=managedidentity`) and optionally for SQL auth to `qfsql`.
- Host storage account: `accountpayableslogicapp`. App Insights: a new, dedicated instance was created (deliberately not reusing `LogicDocuments`' shared one).

## What the workflow does
Trigger: `When a new email arrives (V3)` — polls Office 365 mailbox `Accounting Reports` (`areports@quirchfoods.com`), filtered to messages addressed to the **AP Department** distribution list (`apdepartment@quirchfoods.com`) with attachments.

Flow: looks up a Vendor ID via SQL stored procedure (`[dbo].[spGetVendorIdFromEmail]`), writes qualifying PDF/attachment content to Azure Files (`qfilestoragestandard`, path `/qdocs/Hot Folders/AccountsPayables`), inserts a `DocumentReader` record via SQL, moves the processed email to an "Auto Processed" folder, and sends failure-notification emails (to `rohernandez@quirchfoods.com`, `mnodarse@quirchfoods.com`, `jfigueredo@quirchfoods.com`) at several failure points.

## Important discovery: `apdepartment@quirchfoods.com` is a distribution list
It's a classic mail-enabled DL (`mailEnabled: true`, `securityEnabled: false`, `groupTypes: []` — not a Microsoft 365 Group, not a shared mailbox). It has no inbox and no sign-in identity. Its **sole member** is `Accounting Reports` (`Accounting.Reports@quirchfoods.com`, UPN `areports@quirchfoods.com`) — a real, licensed (E3), hybrid-synced account. This is the actual identity the mailbox-monitoring connection must authenticate as; the `"to": "apdepartment@quirchfoods.com"` filter in the trigger just narrows down which of `areports`' messages arrived via the DL.

## Network path to `qfsql`
`qfsql` lives in `SQL_RG` / `SQL_VNET`, subnet `qcosmos-mi` (EastUS2) — same virtual cluster as `qcosmos-eastus2` (confirmed via matching DNS zone ID `0df600075ba7`).

The Standard app was VNet-integrated to `QSegmentedVnet` / `QSegmentedSubnet2` (already delegated to `Microsoft.Web/serverfarms` and already linked to the shared App Service Plan — no new delegation needed). Path to `qfsql`: `QSegmentedSubnet2` → `FW_HUB_VNET` (hub) → `SQL_VNET`. The hub→SQL_VNET peering has `allowForwardedTraffic: false`, which should in theory block this transit path — but a live test succeeded anyway (`Test-NetConnection -Port 1433` from Kudu returned `True`), most likely because the hub's firewall/NVA performs SNAT before crossing into `SQL_VNET`, making the traffic appear native to the hub rather than "forwarded." Not fully explained, but confirmed working in practice as of 2026-07-09.

**Connector choice:** switched `Get_Vendor_Id` from the managed SQL connector (gateway-based) to the **built-in SQL Server connector**, which uses the app's own VNet integration directly — no on-premises data gateway or "Connection Gateway" selection needed.

## Clone process notes (things that went wrong and why)
1. First clone attempt failed with a generic `InternalServerError` — root cause was a stuck, half-provisioned Standard Logic App named `AccountsPayableDocumentReaderV2` left over from an earlier attempt (state `Stopped`, no linked storage account, cross-resource-group server farm reference). Deleted it and retried.
2. The "Clone to Standard" wizard's destination "Logic app" field is a strict picklist in this build (despite docs implying free-text entry) — it only lists *existing* Standard Logic Apps in the target resource group. Had to pre-create an empty Standard Logic App shell before the wizard would let the clone proceed.
3. Second clone attempt failed with `StandardWorkflowServiceFailure` / `ServiceUnavailable from host runtime` — the destination app's Functions host hadn't finished warming up right after creation. Retried after confirming the site returned HTTP 200 on its own hostname; succeeded.
4. **All managed API connections lose their authorization on clone** — connections get copied as resources but need to be re-authenticated individually in the new app. Broken on this clone: `office365` (3 of 4 "Send an email" actions), `outlook` (the 4th "Send an email" action — a *different* connector type, Outlook.com not Office 365 Outlook, easy to miss), `office365-1` (trigger + attachment handling, real identity `areports@quirchfoods.com`), `azurefile-1` (account-key auth to `qfilestoragestandard`, no managed-identity option available for this connector).

## Remaining work (as of 2026-07-09)
- Re-authenticate all four broken connections listed above.
- Re-test the workflow end-to-end (email trigger → SQL lookup → file write → SQL insert → email move) before re-enabling it (currently disabled post-clone, by design).
- Full security assessment of the new Standard app: see [AccountsPayableDocumentReaderV2-Security-Assessment.md](AccountsPayableDocumentReaderV2-Security-Assessment.md). Priority items: restrict public network access on the storage account (VNet integration already exists, no reason for it to stay open), and audit the unresolved Contributor service-principal grants on `Dev_RG`.
- Once confirmed stable, consider decommissioning the on-premises data gateway dependency on `QSQL` for this specific workflow if no other Logic Apps still rely on it.
