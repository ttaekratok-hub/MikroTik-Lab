# MikroTik Lab — briefing for Claude

This repo is **public**. Never commit secrets, tokens, hostnames, public IPs or
Cloudflare details. Ask the user for those when needed.

## Where Claude works (since 2026-10-09)

- **Repo work happens in Claude Code cloud sessions (claude.ai/code)**, not on
  the Proxmox host. The user moved it off the hypervisor for safety: no
  GitHub token and no project files on the host.
- Cloud sessions can't reach the host or the lab VMs (see the last section).
  The user runs every command themselves in the Proxmox web UI (node → Shell
  for `qm`, VM → Console for the routers and Kali) and sends screenshots.
  Claude reads the screenshots, teaches, and writes the notes in `docs/`.
- Default branch is `main`. Commit lab notes after each milestone and push.
  An old branch `claude/focused-rubin-ek9z57` was merged into `main` via
  PR #1 and can be deleted (ask first).

## Environment

- Proxmox VE 9 on Debian 13 (trixie), single node `themonitor`. About 7 GB
  RAM: keep guest memory in mind before adding VMs.
- The web UI is reachable from the internet only through a Cloudflare Tunnel
  protected by Cloudflare Access. SSH is not exposed.
- Guests: 100 Kali (lab client), 101 CHR1, 102 CHR2, 110 a dev VM that is
  normally stopped during lab work to free RAM. 103 is reserved for CHR3.
- The CHR image is on the host at `/root/chr/chr-7.24.5.img` (needed again
  for CHR3: `qm disk import 103 chr-7.24.5.img local-lvm`).

## Lab status (2026-10-09)

Milestones 1–3 are done; see `README.md` and `docs/01`–`03`.

- CHR1 (VM 101): `ether1` `10.0.12.1/30`. Identity `CHR1`, password set.
  Routes: only connected `10.0.12.0/30` (the milestone 3 static route was
  removed on purpose so OSPF starts clean).
- CHR2 (VM 102): `ether1` `10.0.12.2/30`, `ether2` `192.168.20.1/24`.
  Identity `CHR2`, password set. Only connected routes.
- Kali (VM 100): `eth1` = NetworkManager connection `lab`, static
  `192.168.20.10/24`, no gateway, route `10.0.0.0/8 via 192.168.20.1`.
  `eth0` (profile `Wired connection 1`, locked to `eth0`) is the home
  network and internet; never touch it.
- Expected now: Kali → CHR2 works; Kali → CHR1 times out (CHR1 has no
  return route). **Next: milestone 4, OSPF** (area 0 on CHR1 and CHR2,
  then the ping to CHR1 should work with OSPF routes at distance 110).
- Router passwords live in the user's password manager. Never ask for them.

## Teaching notes

- The user is a CS grad preparing for a Network Operations Engineer I role at
  a small ISP (MikroTik RouterOS, OSPF/BGP/MPLS, DNS/SNMP/RADIUS/TFTP,
  firewalls, Proxmox). Knows theory, new to configuring real routers.
- Ask for a prediction before each test, then explain the result. Analogies
  that worked: bridges are switches, NICs are ports with cables, networks
  are streets, routes are signposts.
- Weak spots from the milestone 2–3 quizzes, worth revisiting: longest
  prefix match (thought the default route wins), the return path (put the
  missing route on the wrong router), a timeout is silence, not a
  "refusal"; ARP is the IP → MAC mapping; traceroute hop counting.
- Mixes up RouterOS and Linux syntax sometimes (`ip route print` on Kali).

## Lab plan

The user is building an OSPF/BGP lab to prepare for an ISP Network Operations
job. Progress and topology are in `README.md`; per-step notes in `docs/`.

- Teach first (2–4 plain sentences on what and why), then give commands. The
  user types every command themselves. One step at a time; wait for their
  output or screenshot. Quiz them after each milestone.
- VM IDs: CHR1 = 101, CHR2 = 102, CHR3 = 103 later. Disks on `local-lvm`.
- Lab bridges `vmbr1`/`vmbr2` stay isolated (no ports, no host IP). Kali
  (VM 100) gets a second NIC on `vmbr2`; never move its `net0`.
- CHR VM settings (checked against docs): see `docs/reference/chr-on-proxmox.md`.
  Free license caps upload at 1 Mbit/s per interface; P1/P10 raise the cap,
  only P-Unlimited removes it.

## How the user works

- Often on an iPhone, using the Proxmox noVNC console in Safari, sometimes
  on a Windows PC with Chrome. In that
  console you can't copy text out, paste doesn't work, there are no arrow keys,
  and long lines are cut off. Keep commands short and never require pasting
  long strings there.
- In Claude Code menus, move with `j`/`k` and confirm with Enter.
- Prefers short, step-by-step instructions and often sends screenshots.

## Safety rules

- Every command the user types on the host runs on the hypervisor as root.
  The user is often remote: a networking mistake cuts off their only access.
- Don't change any of these without explicit approval: host networking
  (`/etc/network/interfaces`, `vmbr*`), the firewall, `cloudflared`, storage.
- List existing guests first (`qm list`, `pct list`). Prefer creating new VMs
  and containers over changing existing ones.
- Screenshots show the user's Proxmox address in the browser bar. Never put
  screenshots or that address in this repo.

## Cloud sessions (claude.ai/code)

Cloud sessions can't reach the host's shell:

- outbound traffic goes through an HTTPS-only proxy without WebSocket support,
  so `cloudflared access ssh` and the web consoles won't work;
- Cloudflare Access blocks unauthenticated requests.

HTTPS API access would need a Cloudflare Access service token plus a Proxmox
API token, stored as environment variables (proposed names:
`CF_ACCESS_CLIENT_ID`, `CF_ACCESS_CLIENT_SECRET`, `PVE_TOKEN_ID`,
`PVE_TOKEN_SECRET`). None of this is set up, and the user prefers giving the commands to
type over giving Claude host access.
