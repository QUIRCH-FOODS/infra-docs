# QSkysheepSQL → qsqlbackups blob storage (private endpoint DNS fix)

**Date:** 2026-06-16
**Subscription:** QuirchFoodsSubscription (2ea66c5f-74a0-4844-801e-e17a7cd5c9b3)

Knowledge record:
- Topic: Private endpoint DNS split-horizon issue for qsqlbackups
- Owning repo: https://github.com/QUIRCH-FOODS/infra-docs
- Canonical guide: https://github.com/QUIRCH-FOODS/infra-docs/blob/main/docs/qskysheepsql-blob-pe-dns.md
- Canonical script/config: https://github.com/QUIRCH-FOODS/infra-docs/blob/main/docs/qskysheepsql-blob-pe-dns.md
- Last verified: 2026-07-21
- Usage intent: Use this for root cause and workaround when Pacific VNET backups cannot reach qsqlbackups private endpoint.

Exception classification:
- Allowed local content under the Knowledge Index Standard as cross-repo operational context and environment-specific DNS behavior.

## Goal
Enable the VM **QSkysheepSQL** to save SQL backups (`BACKUP DATABASE ... TO URL`) to the
storage account **`qsqlbackups`** over a private endpoint.

## Topology
- **QSkysheepSQL** lives in `CBBC_VNET_Pacific / Internal` (10.6.1.0/25); VM IP `10.6.1.133`.
- Storage account `qsqlbackups` has **two private endpoints** (same account, one per region):
  | PE | NIC | IP | VNET / Subnet | Intended consumers |
  |----|-----|----|---------------|--------------------|
  | PE #1 | `Vnet-Pacific-External-PE-nic` | `10.6.1.134` | CBBC_VNET_Pacific / Internal | **QSkysheepSQL** and other Pacific VMs |
  | PE #2 | (EastUS2) | `192.168.114.16` | QuirchVnetEastUS2 / Secure (192.168.114.0/25) | EastUS2 VMs |
- Azure Private DNS zone `privatelink.blob.core.windows.net` correctly holds the PE #1 record
  (`qsqlbackups.privatelink.blob.core.windows.net → 10.6.1.134`) via the PE's `default` DNS zone group.

## Root cause
VMs use the **on-prem/Azure Domain Controller at `192.168.118.132`** (CBBC_Vnet2 / Internal) for DNS,
not Azure's built-in resolver. The DC resolves `qsqlbackups...` to the **EastUS2** PE IP
(`192.168.114.16`) for *every* client. That IP is in QuirchVnetEastUS2 and is **not reachable**
from the Pacific VNET (10.6.1.0/24), so:

- DNS from QSkysheepSQL succeeded (returned `192.168.114.16`) ✅
- `Test-NetConnection qsqlbackups.blob.core.windows.net -Port 443` → `TcpTestSucceeded : False` ❌
  (source `10.6.1.133` → dest `192.168.114.16`, no path between the VNETs)

A single shared DNS zone cannot return different IPs to different VNETs.

## Change applied (immediate fix — hosts file override on QSkysheepSQL)
Added to `C:\Windows\System32\drivers\etc\hosts` on **QSkysheepSQL only**:

```
10.6.1.134   qsqlbackups.privatelink.blob.core.windows.net
10.6.1.134   qsqlbackups.blob.core.windows.net
```

Both names are required: the hosts file does **not** follow CNAMEs, and applications connect to the
public name `qsqlbackups.blob.core.windows.net` (which is normally a CNAME to the `privatelink` name).

### Verification (use these — NOT nslookup)
`nslookup` queries the DNS server directly and **ignores the hosts file**, so it will still show
`192.168.114.16`. Test with tools that go through the Windows DNS Client resolver:

```powershell
ipconfig /flushdns
Resolve-DnsName qsqlbackups.blob.core.windows.net            # expect 10.6.1.134
Test-NetConnection qsqlbackups.blob.core.windows.net -Port 443   # expect TcpTestSucceeded : True
```

(ICMP/ping to the PE NIC is blocked by default — ignore ping failures; the port-443 TCP test is the signal.)

## Trade-offs / status
- The hosts override is **per-machine and manual**. It does not track future PE changes and applies
  only to QSkysheepSQL. Other (EastUS2) VMs keep using `192.168.114.16` via the DC.
- **Long-term fix (not yet implemented):** per-subnet DNS resolution on the DC using Windows
  **DNS Policy zone scopes** — answer `10.6.1.134` to clients in `10.6.1.0/24` and `192.168.114.16`
  to clients in the EastUS2 range. Once in place, remove the hosts entries from QSkysheepSQL.

## SQL Server backup note
`BACKUP TO URL` authenticates with a **SQL Server CREDENTIAL** (SAS token or storage key) scoped to the
container URL — it does not use the VM's Windows identity. Resolution goes through the DNS Client
service, so the hosts override applies. Ensure the storage account's public network access is
Disabled (the PE bypasses the firewall) and the SAS has Read/Write/List/Create on the backup container.
