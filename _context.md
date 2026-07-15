# Infrastructure Context Reference

## Environment Overview
- **Azure Tenant ID:** a87170fd-4ce5-4784-89ee-8be697b9b581
- **Primary Region:** EastUS2
- **Secondary Region:** CentralUS

## Subscriptions
| Name | ID | Purpose |
|------|----|---------|
| [QuirchBI](https://portal.azure.com/) | d7e52ee4-e340-4f24-8509-8460f15da95f | |
| [QuirchDev](https://portal.azure.com/) | b664c182-e0a1-4c35-98be-f1acee3fbfe3 | |
| [QuirchFoodsSubscription](https://portal.azure.com/) | 2ea66c5f-74a0-4844-801e-e17a7cd5c9b3 | |

## Naming Conventions
- Resource Groups: `rg-{workload}-{env}-{region}`
- VNETs: `vnet-{purpose}-{region}`
- Subnets: `snet-{purpose}`

## IP Address Scheme
| VNET / Subnet | CIDR | Purpose |
|---------------|------|---------|
|               |      |         |

## Network Topology (hub-and-spoke)

`FW_HUB_VNET` (ProductionGroup, EastUS2) is the central hub — it has a firewall/NVA and is peered to every spoke below. Peerings marked `allowForwardedTraffic: false` mean traffic transiting through the hub from a third spoke may be blocked at the peering layer unless the hub NVA does SNAT (observed working in practice for at least one path — see qfsql note below).

| Spoke VNet | Resource Group | Subscription | allowForwardedTraffic (hub→spoke) | Notes |
|---|---|---|---|---|
| `QuirchVnetEastUS2` | Default-Networking | QuirchFoodsSubscription | true | Houses VM `QSQL` (192.168.112.37) — legacy on-prem-gateway SQL Server |
| `QSegmentedVnet` | ProductionGroup | QuirchFoodsSubscription | true | Segmented subnets incl. `QSegmentedSubnet2` (delegated to `Microsoft.Web/serverfarms`, used by App Service Plan `ASP-ProductionGroup-9c92` for Standard Logic App VNet integration) and `QSegmentedSubnet3` (VM `QSQL-APT` — a *different* VM from `QSQL`, easy to confuse) |
| `AVD_VNET` | ProductionGroup | QuirchFoodsSubscription | true | AVD session host networking |
| `SQL_VNET` | SQL_RG | QuirchFoodsSubscription | **false** | Hosts SQL MI virtual clusters (`qcosmos-eastus2` subnet `qcosmos-mi`, `qdev-mi` subnet `qdev-mi`). Despite `allowForwardedTraffic: false`, connectivity from `QSegmentedVnet` → hub → here tested successfully (Jul 2026), likely because the hub NVA SNATs before crossing this peering |
| `BI_VNET` / `BI_VNET_2` | BI_RG_PROD | QuirchBI (different subscription) | false | Power BI/Fabric-related; `BI_VNET_2` also has a **VNet data gateway** (`BI_VNET_2-BI-Data-Gateway`) — note this is a Power BI/Fabric-only gateway type and is **not** usable from Logic Apps' managed-connector "Connection Gateway" picker (Logic Apps only supports classic on-prem data gateways there) |
| `QuirchVnet` | default-networking | QuirchFoodsSubscription | true | |
| `QCentralUS-01` | ProductionCentralUS | QuirchFoodsSubscription | true | Central US SQL MI cluster (`qcosmos-centralus`) |
| `ButtsfoodsVN`, `CBBC_VNET`, `CBBC_VNET2` | ButtsfoodsRG, CBBC_RG | QuirchFoodsSubscription | false | Related-company networks peered to the hub |
| `HV_ButtsJackson_...` | RG_ButtsJackson... | external subscription (`6b46ecda-...`) | n/a (peered to `QSegmentedVnet` directly, not via hub) | Cross-tenant/subscription peering |

## SQL Managed Instances
All in EastUS2 unless noted. MIs sharing a DNS zone ID are in the same virtual cluster/subnet.

| MI name | FQDN zone | Resource Group | Subnet |
|---|---|---|---|
| `qcosmos-eastus2` | `0df600075ba7` | SQL_RG (SQL_VNET) | `qcosmos-mi` |
| `qfsql` | `0df600075ba7` (same cluster as above) | SQL_RG (SQL_VNET) | `qcosmos-mi` |
| `qdev-mi` | `8eee4cb5c80c` | SQL_RG (SQL_VNET) | `qdev-mi` |
| `qcosmos-centralus` | `0df600075ba7` | ProductionCentralUS (QCentralUS-01) | `QCosmos-mi-CentralUS` |

## Storage Accounts (key ones — several similarly-named accounts exist, easy to mix up)
| Account | Tier/Redundancy | Purpose |
|---|---|---|
| `qavd` | Premium_LRS (FileStorage) | AVD FSLogix profile shares, EastUS2. Covered by a 2026 Azure Files reservation (see below) |
| `qavdeastus` | PremiumV2_LRS (FileStorage) | AVD FSLogix, EastUS2 region variant. **Not** reservation-eligible (PremiumV2 isn't a supported reservation SKU) |
| `qfilestorage` | Premium_ZRS (FileStorage) | General-purpose file server replacement (RC4→AES remediated 2026-06-27). Covered by a 2024 Azure Files reservation, only ~35% utilized |
| `qfilestoragestandard` | StandardV2_ZRS (FileStorage) | AP document "hot folder" (`/qdocs/Hot Folders/AccountsPayables`) used by the AccountsPayableDocumentReader Logic App |
| `qsftp` | Blob, SFTP feature enabled | Trading-partner/EDI file exchange (Walmart, Coca-Cola, banks, Workday/UKG, etc. — ~26 local SFTP users). Host key is Microsoft-managed per-region, not per-account — see [learn.microsoft.com/azure/storage/blobs/secure-file-transfer-protocol-host-keys](https://learn.microsoft.com/azure/storage/blobs/secure-file-transfer-protocol-host-keys) |

## Azure Reservations
| Reservation | SKU | Qty | Covers | Term | Expires |
|---|---|---|---|---|---|
| `AzureFiles_RI_06-14-2024` | Premium_ZRS_Provisioned_10TB | 3 (30 TiB) | `qfilestorage` | 3yr | 2027-06-14 — re-evaluate size at renewal, usage sustained at ~35% |
| `AzureFiles_RI_07-09-2026` | Premium_LRS_Provisioned_10TB | 2 (20 TiB) | `qavd` | 3yr | 2029-07-09 |

## Key Vault References
| Vault Name | Purpose |
|------------|---------|
| [QDevelopers-Vault](https://portal.azure.com/) | Dev environment |
| [QCosmos-Vault](https://portal.azure.com/) | Production environment |
| JGJKeyVault | QAzureReadOnlyApp client secret storage |

## Lucidchart Diagrams
| Diagram | Link | Last Updated |
|---------|------|--------------|
| Network Architecture | [Lucidchart](https://lucid.app/lucidchart/fb175220-0e8c-4076-8338-006f5dabaa8c/edit?invitationId=inv_63b5344e-398d-42fd-9328-0218d5fc04d9) | |
| Container Apps Infrastructure | [Lucidchart](https://lucid.app/lucidchart/06e595e7-b07c-4233-84ec-39214e799ca1/edit?invitationId=inv_0102c82c-abd8-4ccf-81cd-5bca22dd89d2) | |

## External Services / Vendors
| Service | Purpose | Contact |
|---------|---------|---------|
|         |         |         |

## Azure Databricks
5 workspaces, all Premium tier, EastUS2, all in **QuirchBI** subscription (not QuirchFoodsSubscription): `QDatabricks_WebAuth`, `qf-dbrk-dev`, `qf-dbrk-prod`, `qf-dbrk-uat`, `qf-dbrk-webauth`. DBU spend is stable month to month (~$4,100-4,300/month), dominated by Premium Serverless SQL DBU (~78% of the bill). No DBCU prepurchase in place as of Jul 2026 — Advisor recommends a 50,000 DBCU/1yr (~$46K, ~8% discount) or 150,000 DBCU/3yr (~$135K, ~10% discount) plan sized to this baseline. Note: Databricks prepurchase is **upfront-only billing**, no monthly option, and purchases are final (no cancel/exchange).

## Entra ID App Registrations / Service Principals
| App Name | Purpose | Portal Link |
|----------|---------|-------------|
| QIntuneQueryApp | Service principal to manage Intune | [Azure Portal](https://portal.azure.com/#) |
| QAzureReadOnlyApp | Read-only access to Azure resources and Entra ID for troubleshooting | [Azure Portal](https://portal.azure.com/#) |

---

## QAzureReadOnlyApp — Permissions and Usage

QAzureReadOnlyApp is an Entra ID App Registration with read-only permissions for querying Azure resources and Entra ID objects. Use this service principal for information requests and troubleshooting. For any write or modification operations, use the admin account instead.

### Entra ID Directory Role

| Role | Purpose | Assigned |
|------|---------|----------|
| **Global Reader** | Tenant-wide read-only access equivalent to Global Admin (Exchange, Teams, SharePoint, Security & Compliance, M365 admin center, Entra ID) | Confirmed active as of 2026-07-15 |

### Microsoft Graph API Permissions (Application type)

Expanded 2026-06-05 to cover cross-admin-center read access; all entries below are consented (verified against `appRoleAssignments` 2026-07-15).

| Permission | Purpose |
|------------|---------|
| `Directory.Read.All` | Read users, groups, roles, org info |
| `Device.Read.All` | Read device objects (AVD hosts, Intune devices) |
| `Application.Read.All` | Read app registrations and service principals |
| `AuditLog.Read.All` | Read sign-in and audit logs for troubleshooting |
| `AuditLogsQuery.Read.All` | Read audit log data from all services |
| `Policy.Read.All` | Read Conditional Access and auth policies |
| `User.Read.All` | Read all users' full profiles |
| `Group.Read.All` | Read all groups |
| `AdministrativeUnit.Read.All` | Read all administrative units |
| `Organization.Read.All` | Read organization/tenant info |
| `RoleManagement.Read.Directory` | Read all directory RBAC role assignments |
| `IdentityProvider.Read.All` | Read identity providers |
| `Reports.Read.All` | Read all M365 usage reports |
| `Mail.Read` / `Mail.ReadBasic.All` / `MailboxSettings.Read` / `Calendars.Read` | Exchange Online: read mail, mailbox settings, calendars (all mailboxes) |
| `Sites.Read.All` / `Files.Read.All` | SharePoint/OneDrive: read all site collections and files |
| `Team.ReadBasic.All` / `Channel.ReadBasic.All` / `ChannelMember.Read.All` / `Chat.Read.All` / `ChatMember.Read.All` | Teams admin center: read teams, channels, members, chats |
| `SecurityEvents.Read.All` / `SecurityAlert.Read.All` / `SecurityIncident.Read.All` / `ThreatIndicators.Read.All` | Security & Compliance / Defender: read events, alerts, incidents, threat intel |
| `InformationProtectionPolicy.Read.All` | Read Purview sensitivity labels and label policies |
| `DeviceManagementManagedDevices.Read.All` / `DeviceManagementConfiguration.Read.All` / `DeviceManagementApps.Read.All` / `DeviceManagementServiceConfig.Read.All` | Intune admin center: read managed devices, configuration profiles, apps, service config (added 2026-07-15, mirrors `QIntuneQueryApp`'s read scope) |

### Azure RBAC Role Assignments

Apply to all subscriptions (QuirchBI, QuirchDev, QuirchFoodsSubscription).

| Role | Purpose |
|------|---------|
| Reader | See all resources, configs, properties |
| Monitoring Reader | Read metrics, alerts, diagnostic settings |
| Log Analytics Reader | Query Log Analytics workspaces |
| Key Vault Reader | See vault metadata and access policies (not secrets) |
| Cost Management Reader | Read cost data, budgets, exports, billing info |

### Authentication

Client secret stored in Azure Key Vault: **JGJKeyVault**.

- **Application (client) ID** — `1caf0c34-0866-4db4-b8e3-c6b37f0ee974`
- **Tenant ID** — `a87170fd-4ce5-4784-89ee-8be697b9b581`
- **Client secret** — retrieve from JGJKeyVault

> ⚠️ Do not store the secret in GitHub repos or AI memory. Retrieve from JGJKeyVault and provide in-session as needed.

### What QAzureReadOnlyApp CAN do

- List/inspect all Azure resources (VMs, VNETs, NSGs, SQL MI, Container Apps, etc.)
- Read VNET configs, peering, subnets, route tables
- Read NSG rules and effective security rules
- Azure Monitor metrics, alerts, diagnostic settings
- Query Log Analytics workspaces
- Read Key Vault metadata (not secret values)
- Read cost data, budgets, exports, billing
- Query Entra ID: users, groups, devices, app registrations, sign-in logs, admin units, RBAC role assignments
- Read Conditional Access policies
- Read Exchange Online mail, mailbox settings, calendars (Global Reader + Graph mail permissions)
- Read Teams: teams, channels, channel/chat members, chat messages
- Read SharePoint/OneDrive site collections and files
- Read Security & Compliance: alerts, incidents, threat indicators, Purview sensitivity labels
- Read M365 usage reports and audit logs across services
- Read Intune: managed devices, configuration profiles, apps, service config

### What QAzureReadOnlyApp CANNOT do (use admin account)

- Network Watcher actions (packet capture, connection troubleshoot, topology) — no read-only built-in role exists; minimum is Network Contributor (write)
- Create, modify, or delete any Azure resource
- Modify NSG rules, route tables, VNET peerings
- Read Key Vault secret values
- Deploy Bicep/ARM templates
- Manage role assignments or Entra ID objects
