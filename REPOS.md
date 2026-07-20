# Repository Index

Central manifest of every infrastructure repo (QUIRCH-FOODS org) and the personal
PowerShell script library. Use this to find which repo owns a given domain so you can
**reference, upgrade, or update** it from a session without hunting.

> **How to use in a session:** name the infrastructure or script you're after; find the
> owning repo below; then clone/pull that repo (or the specific path) to work on it.
> This `infra-docs` repo is the hub — domain detail lives in the linked repos.

Last reviewed: 2026-07-20.

---

## QUIRCH-FOODS org repos

| Repo | Scope | Key contents / entry points |
|------|-------|------------------------------|
| [infra-docs](https://github.com/QUIRCH-FOODS/infra-docs) | **This hub** — central docs, architecture diagrams, standards, environment grounding | [`README.md`](README.md), [`_context.md`](_context.md), [`REPOS.md`](REPOS.md), `docs/`, `diagrams/` |
| [azure-networking](https://github.com/QUIRCH-FOODS/azure-networking) | Azure Virtual WAN, VNETs, peering, DNS, firewall | VNET/peering configs, DNS + firewall rules |
| [onprem-network](https://github.com/QUIRCH-FOODS/onprem-network) | On-prem core switch configs, DHCP, VLAN, snooping tables | Cisco switch configs; see also infra-docs `docs/cisco-switch-troubleshooting.md` |
| [azure-sql-mi](https://github.com/QUIRCH-FOODS/azure-sql-mi) | Azure SQL Managed Instance, MI Link, DR, Key Vault connection templates | MI Link + DR templates, KV connection templates |
| [af-backup-ops-eastus2](https://github.com/QUIRCH-FOODS/af-backup-ops-eastus2) | Daily internal-only copy of SQL MI backups: `qfsql-daily` → `qfsql-daily-immutable` | Flex Consumption Function, private endpoints, MI auth |
| [Azure-BI-Infra](https://github.com/QUIRCH-FOODS/Azure-BI-Infra) | BI / PowerBI: Databricks env, private-link connections, permissions | Databricks env specs, private-link, RBAC |
| [Azure-Cost](https://github.com/QUIRCH-FOODS/Azure-Cost) | Monthly cost analysis — executive summary + savings opportunities | Cost reports / savings analysis |
| [AzStorage-Audit](https://github.com/QUIRCH-FOODS/AzStorage-Audit) | Fleet-wide Azure Storage change-audit logging (who created/modified/deleted blobs) | Storage audit logging for prod storage accounts |
| [azure-container-apps](https://github.com/QUIRCH-FOODS/azure-container-apps) | Azure Container Apps environments and deployments | Container Apps env configs, deployments |
| [azure-apim](https://github.com/QUIRCH-FOODS/azure-apim) | API Management policies, OAuth 2.0 / Entra ID integration | APIM policies, OAuth/Entra integration |
| [azure-avd](https://github.com/QUIRCH-FOODS/azure-avd) | AVD session host configs, FSLogix, Intune enrollment | `scripts/Get-AVDPersonalPoolRestartAssignments.ps1`, `configs/`, `docs/personal-pool-restart-permissions.md` |
| [avd-automation](https://github.com/QUIRCH-FOODS/avd-automation) | AVD ops automation for `aa-avd-ops-eastus2` | `fslogix-profile-handlers/` (Reset-AVDUserProfile), `drain-mode/` (Set-AVDDrainMode) |
| [helpdesk-runbook-portal](https://github.com/QUIRCH-FOODS/helpdesk-runbook-portal) | Web portal for helpdesk to run Automation runbooks | Entra SSO, per-script group-based authz |
| [intune-management](https://github.com/QUIRCH-FOODS/intune-management) | Intune device mgmt, config profiles, compliance, enrollment | Config profiles, compliance policies, enrollment scripts |
| [AzureMicrosoft365](https://github.com/QUIRCH-FOODS/AzureMicrosoft365) | Microsoft 365 configuration | M365 tenant configuration |
| [alerts-enrichment](https://github.com/QUIRCH-FOODS/alerts-enrichment) | Pre-explain failures before they hit the Teams alert channel | Alert enrichment / analysis automation |

---

## Personal scripts — `jgarrote83/MyPersonalScripts`

**Not in the QUIRCH-FOODS org** (personal account: [github.com/jgarrote83/MyPersonalScripts](https://github.com/jgarrote83/MyPersonalScripts)).
The working library of reusable PowerShell — check here first before writing a new script.
**346 scripts** across three top-level categories. Category map with notable/reusable scripts:

### `Azure/` (91 scripts)
| Folder | Count | What's there | Notable scripts |
|--------|-------|--------------|-----------------|
| `Azure/General/` | 28 | VM lifecycle, disks, VNET, encryption, VPN, AVD helpers | `Troubleshoot-AVD.ps1`, `Force-AVDUserLogoff.ps1`, `AVD/AvdProfilesCleanUP/Cleanup-AVDProfiles.ps1`, `AVD/Install-SnippingTool-AVD.ps1` (Snipping Tool detect/repair/install), `Backup And Restore of VMs.ps1`, `Change-NSG-on-VM-NIC.ps1` |
| `Azure/Azure-Functions/` | 7 | Function-app connectivity/SQL test harness | `Test-SQLConnection.ps1`, `Test-NetworkConnectivity.ps1`, `Deploy-SQLConnectionTest.ps1` |
| `Azure/General/Entra ID/` | — | Entra ID reporting / app mappings | `Get-EntraIDLastLogin.ps1`, `EnterpriseApplications/Export-EnterpriseAppAttributeMappings-CLI.ps1` |

### `Domain-Administration/` (161 scripts)
| Folder | Count | What's there | Notable scripts |
|--------|-------|--------------|-----------------|
| `Computer-Management/` | 36 | Remote PC ops, OU moves (RoboMove*), Teams-on-AVD installers, OS/inventory | `Install-Teams-AVD*.ps1`, `Install-MSI-To-RemotePC.ps1`, `Get-LastLogon_Working.ps1`, `Rejoin computer to the domain.ps1` |
| `Reports/` | 32 | AD/AV/DFS/WSUS/.NET reporting to email | `Get-Antivirus.ps1`, `DFS Report and send it to email Final.ps1`, `WSUS Report To Email.ps1`, `Find useraccounts inactive by days...ps1` |
| `User-Management/` | 26 | Bulk AD attribute edits, proxy addresses, disable/move | `Add-AD-Attributes-From-Excel.ps1`, `Unlock User Account.ps1`, `ExportUsersWithEmailAddress-Name-EmployeeID.ps1` |
| `Group-Management/` | 15 | AD group membership copy/export, FGPP dynamic groups | `Copy AD Group members between groups..ps1`, `FPGG Dynamic Group/Manage-FGPPGroupMembership.ps1` |
| `RDS/` | 13 | RDS session logoff / profile info | `Logoff-RemoteUser.ps1`, `RDS_LogoffDisconnectedUsers.ps1` |
| `General/` | 12 | FGPP permissions, storage sense, event logs, updates | `Configure-FGPPPermissions.ps1`, `Enforce Windows updates.ps1` |
| `Networking/` | 6 | Connectivity tests, SonicWall API, wireless profiles | `Test connectivity to Host (ICMP and Ports).ps1`, `SonicOS-API-Configuration-Guide.md`, `sonicwall.ps1` |
| `Bulk-Operations/` | 5 | Bulk AD user import from CSV/Excel | `ImportBulkUsersToAD.ps1` |
| `Service-Accounts/` | 3 | gMSA create/verify, proxy addresses | `Create gMSA account in an OU.ps1`, `Verifying gMSA account.ps1` |
| `SQL-Server/` | 2 | SQL VHD best practices | `SQL VHD Creation best practices.ps1` |
| `HyperV/` | 2 | Trusted hosts for Hyper-V | `AddHyperVServertotrustedHosts.ps1` |
| `Troubleshooting/` | 2 | DC health / AD replication | `AzureComprehensive Health Check for Domain Controllers.ps1`, `Test-ADReplication.ps1` |
| `DFSR/` | 1 | DFSR backlog commands | `DFS_Backlog_QF_Commands.ps1` |

### `Microsoft-365/` (94 scripts)
| Folder | Count | What's there | Notable scripts |
|--------|-------|--------------|-----------------|
| `General/` | 24 | Cloud-only groups, DLs without proxy, SMTP tests, universal print | `Get-CloudOnlyGroups.ps1`, `Install-UniversalPrinter.ps1`, `Testing Office365 SMTP configuration.ps1` |
| `Mailbox/` | 21 | Shared/convert mailbox, forwarding-rule blocks, holds, size | `Convert Mailbox to Shared mailbox.ps1`, `Disable Forwarding rules Capabilities in Office365.ps1`, `putmailboxesonhold.ps1` |
| `Distribution-Groups/` | 18 | DL membership add/remove/export, GAL hide, DL→security | `Sync-DistributionList.ps1`, `Hide Distribution Group from GAL (Global address list).ps1` |
| `Intune/` | 8 | Intune policy apply/sync/troubleshoot, AI-assisted | `Invoke-IntunePolicyTroubleshooter.ps1`, `Manage-IntunePolicies.ps1`, `Get-IntuneAppliedPolicies.ps1` |
| `Licensing/` | 7 | License assign/export, Fabric free, SKU inventory | `Get-LicensingInventory.ps1`, `Assign-FabricFreeLicense.ps1` |
| `Exchange/` | 6 | EXO cloud-only groups, Jira app-access policy | `Create-JiraApplicationAccessPolicy.ps1`, `Get-EXOCloudOnlyGroups.ps1` |
| `AI-Integration/` | 1 | Message-trace / quarantine analyzer | `Exchange-MessageTrace-Quarantine-Analyzer.ps1` |
| `MFA/` | 1 | Bulk MFA enablement | `USetMFA per bulk.ps1` |
| `User-Management/` | 1 | Set M365 user photo | `Add photo 365.ps1` |

> Counts and notable scripts as of 2026-07-20. When scripts are added/renamed, refresh
> the counts (`find <category> -name '*.ps1' | wc -l`) and add any new notable entries.
