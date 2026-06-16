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

## Related Repos
- azure-networking
- azure-avd
- azure-sql-mi
- azure-container-apps
- azure-apim
- onprem-network
- intune-management

## Infrastructure Context
See [`_context.md`](./_context.md) for the environment grounding reference.
