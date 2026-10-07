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
- `tmux` and `git` may not be installed yet; check with `which tmux git`.

## Status (2026-10-07)

Claude Code is installed but **not signed in**. A sign-in attempt from the
user's phone failed and may have left these behind:

- background processes from `tail -f ~/c | claude auth login`
- files `~/c` and `~/l`
- the sign-in link saved in the `kali` node's Notes

Clean up:

```bash
pkill -x tail; pkill -f 'claude auth'; rm -f ~/c ~/l
pvesh set /nodes/kali/config --delete description   # only if Notes still holds the sign-in link
```

## Next steps

1. Sign in from a computer, where copy and paste work: run `claude`, open the
   link, paste the code back. Remote Control needs a claude.ai subscription
   login (Pro/Max/Team/Enterprise); API keys don't work for it.
2. Start Remote Control in tmux so it survives closing the browser tab:
   `tmux new -s mikrotik`, then `cd ~/MikroTik-Lab && claude remote-control --name MikroTik`,
   then detach with Ctrl+b, d. tmux sessions don't survive a reboot.
3. Build the CHR VM (below).

## CHR VM plan

Check the current MikroTik CHR docs before running anything.

- Get the CHR raw disk image (`chr-<version>.img.zip`) for current stable
  RouterOS v7 from mikrotik.com/download.
- Unzip it, import it as the VM disk (`qm disk import`), and make it the boot
  disk. Confirm which disk bus and NIC model CHR supports instead of assuming.
- A small VM is enough: 1 vCPU, about 1 GB RAM, NIC on `vmbr0`.
- The free CHR license caps upload at 1 Mbit/s per interface. That's fine for a
  lab; trial or paid licenses remove the cap.
- Before creating anything, ask for the VM ID, storage (e.g. `local-lvm`) and
  bridge. Also ask whether CHR is only a lab router or should route real
  traffic.

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
