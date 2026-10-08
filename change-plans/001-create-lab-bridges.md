# Change plan 001: create isolated lab bridges `vmbr1` and `vmbr2`

| | |
|---|---|
| Date | 2026-10-07 |
| System | Proxmox host (single node) |
| Risk | Low impact, but it touches host networking: an apply reloads all interfaces |
| Result | ✅ Completed, verified |

## Purpose

Create two virtual switches for the lab: `vmbr1` for the CHR1–CHR2
point-to-point link and `vmbr2` for the CHR2–Kali LAN. They must be isolated
from the home network.

## Pre-checks

- `ip -br link show type bridge`: only `vmbr0` exists.
- Remote access depends on `vmbr0`, so it must not change.

## Steps

1. Back up the config: `cp /etc/network/interfaces /root/interfaces.bak`
2. Node → System → Network → Create → Linux Bridge → `vmbr1` (no IP, no
   ports, Autostart on)
3. Same for `vmbr2`
4. Read *Pending changes*: only additions, nothing touching `vmbr0`
5. Apply Configuration

## Verification

- `vmbr1` and `vmbr2` show Active: Yes
- `vmbr0` keeps its IP and gateway; the web UI stays reachable
- The task `SRV networking - Reload` shows OK

## Rollback

If `vmbr0` breaks and a shell is still available:

```
cp /root/interfaces.bak /etc/network/interfaces
ifreload -a
```

If remote access is lost, use the local console on the server.

## Outcome

Applied with no errors. The rollback wasn't needed.
