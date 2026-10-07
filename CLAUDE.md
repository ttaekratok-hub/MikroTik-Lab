# MikroTik Lab — briefing for Claude

This repo is **public**. Never commit secrets, tokens, hostnames, public IPs or
Cloudflare details. Ask the user for those when needed.

## Goal

- Run MikroTik RouterOS as a VM (**CHR**, Cloud Hosted Router) on the user's
  home Proxmox server.
- Claude Code runs on the Proxmox host; the user drives it from the Claude
  mobile app through **Remote Control**.
- Later: separate Claude sessions for other projects (e.g. trading, work), each
  in its own folder and tmux session.

## Environment

- Proxmox VE 9 on Debian 13 (trixie), `-pve` kernel 7.0.x. Node name is `kali`;
  despite the name it is not Kali Linux.
- The web UI is reachable from the internet only through a Cloudflare Tunnel
  protected by Cloudflare Access. SSH is not exposed through Cloudflare. The
  user gets a shell through the web UI (node → Shell).
- No Proxmox subscription. The enterprise repos (`pve`, `ceph-squid`) return 401,
  which makes `apt update` exit non-zero and stops `&&` chains. Fix in the GUI:
  node → Updates → Repositories → disable the enterprise entries, Add →
  No-Subscription. Not confirmed done yet.
- Claude Code is installed for root with the native installer
  (`~/.local/bin/claude`, PATH added in `/root/.bashrc`).

## Status (2026-10-07)

- Claude Code is signed in and Remote Control works: sessions run on the host
  itself as root (not in the cloud). It is not running inside tmux yet.
- `tmux` and `git` are installed. This repo is cloned at `/root/repos/MikroTik-Lab`.
- Enterprise apt repos are still enabled (no-subscription repo not added). The
  Ceph no-subscription repo is not needed; just disable the Ceph enterprise one.
- Old sign-in leftovers (`~/c`, `~/l`, stray `tail`/`claude auth` processes,
  sign-in link in the node's Notes) were not checked; the user should clean
  them up themselves.
- Lab milestone 1, step 1 done: isolated bridges `vmbr1`, `vmbr2` created.

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

- Often on an iPhone, using the Proxmox noVNC console in Safari. In that
  console you can't copy text out, paste doesn't work, there are no arrow keys,
  and long lines are cut off. Keep commands short and never require pasting
  long strings there.
- In Claude Code menus, move with `j`/`k` and confirm with Enter.
- Prefers short, step-by-step instructions and often sends screenshots.

## Safety rules

- This is the hypervisor and Claude runs as root. The user is often remote: a
  networking mistake cuts off their only access.
- Don't change any of these without explicit approval: host networking
  (`/etc/network/interfaces`, `vmbr*`), the firewall, `cloudflared`, storage.
- List existing guests first (`qm list`, `pct list`). Prefer creating new VMs
  and containers over changing existing ones.
- Keep permission prompts on; never use bypass modes.
- Suggest moving unrelated projects, such as trading, into their own LXC
  container.

## Cloud sessions (claude.ai/code)

Cloud sessions can't reach the host's shell:

- outbound traffic goes through an HTTPS-only proxy without WebSocket support,
  so `cloudflared access ssh` and the web consoles won't work;
- Cloudflare Access blocks unauthenticated requests.

HTTPS API access would need a Cloudflare Access service token plus a Proxmox
API token, stored as environment variables (proposed names:
`CF_ACCESS_CLIENT_ID`, `CF_ACCESS_CLIENT_SECRET`, `PVE_TOKEN_ID`,
`PVE_TOKEN_SECRET`). None of this is set up. Do host work through the Remote
Control session on the host.
