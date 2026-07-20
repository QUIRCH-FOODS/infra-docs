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

## This repo is the hub
`infra-docs` is the central index for the Quirch Foods environment. Each infrastructure
domain lives in its own repo; the reusable PowerShell library lives in a personal repo.
When you need to **reference, upgrade, or update** a specific project, start here:

➡️ **[REPOS.md](REPOS.md)** — full repository manifest:
- **QUIRCH-FOODS org repos** with scope + key entry points (networking, SQL MI, AVD, Intune, M365, container apps, cost/audit, etc.)
- **`jgarrote83/MyPersonalScripts`** (personal account, not in the org) — 346 PowerShell scripts by category, with notable/reusable scripts called out so you know what already exists before writing new.

To work on a domain: find its repo in REPOS.md → clone/pull that repo (or the specific path) → make the change there.

## Runbooks
- [Cisco switch troubleshooting](docs/cisco-switch-troubleshooting.md) — SSH access to the CBB/Quirch Cisco switches via the read-only `troubleshoot` account (parser view), plink usage, gotchas, and diagnostic commands.

## Infrastructure Context
See [`_context.md`](./_context.md) for the environment grounding reference.
