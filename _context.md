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
| QuirchVnetEastUS2 / GatewaySubnet | 192.168.115.192/26 | |
| QuirchVnetEastUS2 / Default | 192.168.112.0/24 | |
| QuirchVnetEastUS2 / External | 192.168.114.128/25 | |
| QuirchVnetEastUS2 / Secure | 192.168.114.0/25 | Holds `qsqlbackups` PE #2 (192.168.114.16) for EastUS2 VMs |
| AVD_VNET / Internal_Subnet | 192.168.120.0/24 | |
| AVD_VNET / External_Subnet | 192.168.121.0/24 | |
| SQL_VNET / Internal_Subnet | 192.168.124.0/25 | |
| SQL_VNET / SQLManageInstances1 | 192.168.124.128/25 | |
| SQL_VNET / WebAppServices | 192.168.125.0/25 | |
| CBB_Vnet / CBBC_External | 192.168.117.128/26 | |
| CBB_Vnet / CBBC_Internal | 192.168.117.0/25 | |
| CBBC_Vnet2 / GatewaySubnet | 192.168.118.192/26 | |
| CBBC_Vnet2 / Internal | 192.168.118.0/25 | Domain Controller / DNS at 192.168.118.132 |
| CBBC_Vnet2 / External | 192.168.118.128/25 | |
| ButtfoodsVN / Internal | 192.168.116.0/25 | |
| ButtfoodsVN / External | 192.168.116.128/25 | |
| CBBC_VNET_Pacific / Internal | 10.6.1.0/25 | QSkysheepSQL VM (10.6.1.133); `qsqlbackups` PE #1 (10.6.1.134) |
| CBBC_VNET_Pacific / External | 10.6.1.128/26 | |
| CBBC_VNET_Pacific / GatewaySubnet | 10.6.1.192/26 | |
| QuirchVnet / QuirchVMAzure | 192.168.104.0/23 | |
| QuirchVnet / GatewaySubnet | 192.168.107.224/27 | |

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

## Entra ID App Registrations / Service Principals
| App Name | Purpose | Portal Link |
|----------|---------|-------------|
| QIntuneQueryApp | Service principal to manage Intune | [Azure Portal](https://portal.azure.com/#) |
| QAzureReadOnlyApp | Read-only access to Azure resources and Entra ID for troubleshooting | [Azure Portal](https://portal.azure.com/#) |

---

## QAzureReadOnlyApp — Permissions and Usage

QAzureReadOnlyApp is an Entra ID App Registration with read-only permissions for querying Azure resources and Entra ID objects. Use this service principal for information requests and troubleshooting. For any write or modification operations, use the admin account instead.

### Microsoft Graph API Permissions (Application type)

| Permission | Purpose |
|------------|---------|
| `Directory.Read.All` | Read users, groups, roles, org info |
| `Device.Read.All` | Read device objects (AVD hosts, Intune devices) |
| `Application.Read.All` | Read app registrations and service principals |
| `AuditLog.Read.All` | Read sign-in and audit logs for troubleshooting |
| `Policy.Read.All` | Read Conditional Access and auth policies |

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

- **Application (client) ID** — from the App Registration overview
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
- Query Entra ID: users, groups, devices, app registrations, sign-in logs
- Read Conditional Access policies

### What QAzureReadOnlyApp CANNOT do (use admin account)

- Network Watcher actions (packet capture, connection troubleshoot, topology) — no read-only built-in role exists; minimum is Network Contributor (write)
- Create, modify, or delete any Azure resource
- Modify NSG rules, route tables, VNET peerings
- Read Key Vault secret values
- Deploy Bicep/ARM templates
- Manage role assignments or Entra ID objects
