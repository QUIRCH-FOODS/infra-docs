# infra-docs

## Overview
Central infrastructure documentation, architecture diagrams, and standards for the Quirch Foods environment.

## Architecture
- Link to Lucidchart diagram: [Diagram Title](LUCID_LINK_HERE)
- Exported diagram: see `diagrams/` folder

## Current State
- Subscription(s):
- Resource Group(s):
- Key Resources:

## Design Decisions
| Decision | Rationale | Date |
|----------|-----------|------|
|          |           |      |

## Known Issues / Gotchas
- **`qsqlbackups` blob storage has two private endpoints** (one per region/VNET): `10.6.1.134` in CBBC_VNET_Pacific and `192.168.114.16` in QuirchVnetEastUS2/Secure. The shared DC DNS (192.168.118.132) resolves `qsqlbackups.privatelink.blob.core.windows.net` to the **EastUS2** IP (`192.168.114.16`) for everyone, which is **unreachable from the Pacific VNET**. The Pacific VM **QSkysheepSQL** is pinned to `10.6.1.134` via a hosts-file override — see [docs/qskysheepsql-blob-pe-dns.md](docs/qskysheepsql-blob-pe-dns.md). Proper long-term fix is per-subnet DNS resolution (Windows DNS Policy zone scopes) on the DC.

## Related Repos (QUIRCH-FOODS org)
This repo is the central index. Other infrastructure domains live in their own repos — consult the relevant one for domain detail:

| Repo | Scope |
|------|-------|
| [infra-docs](https://github.com/QUIRCH-FOODS/infra-docs) | Central infra documentation, architecture diagrams, standards (this repo) |
| [azure-networking](https://github.com/QUIRCH-FOODS/azure-networking) | Azure Virtual WAN, VNETs, peering, DNS, firewall |
| [onprem-network](https://github.com/QUIRCH-FOODS/onprem-network) | On-prem core switch configs, DHCP, VLAN, snooping tables |
| [azure-sql-mi](https://github.com/QUIRCH-FOODS/azure-sql-mi) | Azure SQL Managed Instance, MI Link, DR, Key Vault connection templates |
| [Azure-BI-Infra](https://github.com/QUIRCH-FOODS/Azure-BI-Infra) | BI / PowerBI: Databricks env, private-link connections, permissions |
| [azure-container-apps](https://github.com/QUIRCH-FOODS/azure-container-apps) | Azure Container Apps environments and deployments |
| [azure-apim](https://github.com/QUIRCH-FOODS/azure-apim) | API Management policies, OAuth 2.0 / Entra ID integration |
| [azure-avd](https://github.com/QUIRCH-FOODS/azure-avd) | AVD session host configs, FSLogix, Intune enrollment |
| [avd-automation](https://github.com/QUIRCH-FOODS/avd-automation) | AVD ops automation: Reset-AVDUserProfile runbook, helpdesk wrapper (aa-avd-ops-eastus2) |
| [helpdesk-runbook-portal](https://github.com/QUIRCH-FOODS/helpdesk-runbook-portal) | Web portal for helpdesk to run Automation runbooks; Entra SSO, group-based authz |
| [intune-management](https://github.com/QUIRCH-FOODS/intune-management) | Intune device mgmt, config profiles, compliance, enrollment |
| [AzureMicrosoft365](https://github.com/QUIRCH-FOODS/AzureMicrosoft365) | Microsoft 365 configuration |

## Infrastructure Context
See [`_context.md`](./_context.md) for the environment grounding reference.
