# Cisco switch troubleshooting — SSH access & read-only VIEW

**Date:** 2026-07-06

Knowledge record:
- Topic: Cisco switch troubleshooting (read-only access)
- Owning repo: https://github.com/QUIRCH-FOODS/infra-docs
- Canonical guide: https://github.com/QUIRCH-FOODS/infra-docs/blob/main/docs/cisco-switch-troubleshooting.md
- Canonical script/config: https://github.com/QUIRCH-FOODS/infra-docs/blob/main/docs/cisco-switch-troubleshooting.md
- Last verified: 2026-07-21
- Usage intent: Use this when you need safe read-only SSH diagnostics on Cisco switches.

Exception classification:
- Allowed local content under the Knowledge Index Standard as an operational deep-dive and environment-specific troubleshooting note.

How to connect to the Quirch/CBB Cisco switches from a Windows admin machine and run
read-only troubleshooting, plus the IOS `parser view` that provides the restricted account.

## Read-only account
A dedicated **read-only** local account is provisioned on each switch via an IOS CLI
**parser view** named `READONLY`. It permits only `show`, `ping`, `traceroute`,
`terminal length/width`, and `exit` — no config, no `debug`.

- **Username:** `troubleshoot`
- **Password / view secret:** `Tlook01`
- The account is safe for automated/AI-driven troubleshooting: it cannot change config.

> Credentials are stored here for internal ops convenience. Rotate `Tlook01` if it is ever
> exposed outside the team, and update this doc.

## Known devices
| Device | Mgmt IP | Model | Notes |
|--------|---------|-------|-------|
| `CBBC-HOU-3850-DS01` | `10.5.21.1` | 2× WS-C3850-24P stack, IOS-XE 03.07.04E | Distribution stack (HOU) |
| `CBBC-HOU-3850-AS01` | (via CDP) | WS-C3850 | Access, on DS01 Gi1/1/1 |
| `CBBC-HOU-3850-AS02` | `10.5.21.3` | WS-C3850, IOS 15.2(2)E8 | Access, on DS01 Gi1/1/2 |
| `CBBC-HOU-SW-AS03` | (via CDP) | C9200L-24 | Access, on DS01 Gi2/1/3 |
| `CBBC-HOU-3850-AS04` | (unknown) | — | Access, on DS01 Gi1/1/3 (**link down as of 2026-07-06**) |

Mgmt IPs are in the `10.5.21.0/24` range. Use `show cdp neighbors detail` from a reachable
switch to discover neighbor mgmt IPs.

## How to connect (Windows)
Both clients ship on the admin machine:
- OpenSSH: `C:\Windows\System32\OpenSSH\ssh.exe`
- PuTTY plink: `C:\Program Files\PuTTY\plink.exe`

Use **plink** for scripted/non-interactive runs because it can pass the password on the CLI
and cache the host key non-interactively. (Built-in OpenSSH cannot take a password as an
argument, which is why plink is preferred here.)

### 1. Test reachability first
Switch mgmt interfaces usually **block ICMP** — test the TCP port, not ping:
```powershell
Test-NetConnection 10.5.21.1 -Port 22 -WarningAction SilentlyContinue
# TcpTestSucceeded : True  (PingSucceeded may be False — that's normal)
```

### 2. Run commands via plink
Put the commands in a text file and pipe it into plink's **stdin** (an interactive shell
channel). Do **not** use plink `-m` — it concatenates the file into a single SSH exec
command that IOS rejects (`Line has invalid autocommand`).

```powershell
Get-Content .\cmds.txt -Raw |
  & "C:\Program Files\PuTTY\plink.exe" -ssh -batch troubleshoot@10.5.21.1 -pw "Tlook01"
```

`cmds.txt` example:
```
noop-absorb
terminal length 0
show interfaces status
show cdp neighbors detail
exit
```

### First-connect host key
First connection needs the host key cached. `-hostkey *` is **not** valid syntax. Simplest:
run one throwaway command interactively once and accept the key, or run with `-batch` — the
key gets cached after the first successful auth. (`plink: '*' is not a valid format` means you
used `-hostkey *`; drop it.)

## Gotchas that WILL bite you (learned the hard way)
1. **First character of every line is eaten.** The switch consumes the first keystroke of the
   session, so the first real command gets mangled (`terminal length 0` → `erminal...`, and you
   see stray `o;?` / `o;` at the top). **Fix:** make the *first* line of `cmds.txt` a
   throwaway (e.g. `noop-absorb`) that you don't mind failing.
2. **Paging (`--More--`) truncates output.** `terminal length 0` must actually apply — because
   of gotcha #1 the first one often gets eaten, so **send it twice** (after the throwaway line),
   or rely on the throwaway absorbing the eaten char.
3. **stderr looks scary but isn't.** PowerShell wraps plink's `Keyboard-interactive
   authentication prompts from server` stderr in red `NativeCommandError` text. Auth still
   succeeded — check for real command output below it.
4. **PowerShell has no heredoc.** `<<'EOF'` fails (`The '<' operator is reserved`). Use a temp
   file + `Get-Content -Raw | plink`.
5. **READONLY view strips `show run interface`.** It returns only `end` (5 bytes). Use the
   specific `show` commands (`show interfaces ...`, `show interfaces ... switchport`) instead of
   `show running-config`.
6. **Transient drops.** `FATAL ERROR: Network error: Software caused connection abort` on
   connect is transient — just retry.

## Useful read-only diagnostic commands
| Goal | Command |
|------|---------|
| Device/stack/uptime/reload reason | `show version` |
| Hardware/SFP inventory & serials | `show inventory` |
| PSU / fan / temperature | `show env all` |
| Port up/down at a glance | `show interfaces status` |
| One interface detail + counters | `show interfaces Gi1/1/3` |
| L2 config of a port (trunk/native/allowed) | `show interfaces Gi1/1/3 switchport` |
| SFP model / type | `show idprom interface Gi1/1/3` |
| SFP light levels (Tx/Rx) | `show interfaces Gi1/1/3 transceiver detail` — *OEM SFPs report "Diagnostic Monitoring is not implemented"* |
| Error counters | `show interfaces Gi1/1/3 counters errors` |
| Neighbor map + neighbor mgmt IPs | `show cdp neighbors [detail]` |
| Interface event history | `show logging \| include 1/1/3` |

## Reading a "notconnect" fiber trunk
- `notconnect` on a fiber port = **no light received** (physical / L1), not a config problem.
- **CDP is the fast splitter:** CDP works even across trunk/native-VLAN mismatches, so if a
  port has **no CDP neighbor AND no link**, the fault is physical (dead far end, broken/dirty
  fiber, swapped Tx/Rx, or bad SFP) — not L2 config.
- Check the far-end switch's matching port (`show interfaces ... status`, its log, its CDP).
  `admin down` on either end = someone shut it; `notconnect` both ends = fiber/SFP.
- **OEM (non-Cisco) SFPs** (e.g. `SFPGLCSXMMST`, vendor `OEM`) work but report **no DOM**, so
  you can't read optical power to confirm light — swap-test with a known-good SFP instead.

## Creating the READONLY view (run once per switch, as a privileged admin)
Requires an admin login with a level-15 / enable secret set (`enable view` prompts for it).
```
enable view
configure terminal
!
parser view READONLY
 secret Tlook01
 commands exec include all show
 commands exec include ping
 commands exec include traceroute
 commands exec include terminal length 0
 commands exec include terminal width 0
 commands exec include exit
!
username troubleshoot view READONLY secret Tlook01
end
write memory
```
`aaa new-model` must be enabled for parser views to take effect. After creation, verify with
`show parser view` and by logging in as `troubleshoot`.
