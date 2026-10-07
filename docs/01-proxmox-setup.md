# Milestone 1: Proxmox setup

Goal: build the lab's "physical" layer inside Proxmox: two isolated virtual
switches, two CHR router VMs, and a lab NIC on Kali.

## Step 0: record the current state (2026-10-07)

Before changing anything, check what exists, so you can tell later whether a
change broke something.

```
qm list
pvesm status
ip -br link show type bridge
```

What I found:

- One VM: 100 (Kali), with one NIC (`net0`) on `vmbr0`.
- Storage: `local` (directory) and `local-lvm` (LVM-thin, for VM disks).
- One real bridge: `vmbr0` (home LAN and my remote access).
  `fwbr100i0` is a helper bridge Proxmox creates for Kali's firewall.

Decisions: routers get VM IDs **101** (CHR1) and **102** (CHR2), CHR3 later
gets 103. Disks go on `local-lvm`. Kali gets a **second** NIC on `vmbr2`, so
its normal network access is unchanged.

## Step 1: create isolated bridges `vmbr1` and `vmbr2` ✅

**What a Linux bridge is:** a software switch. It works at Layer 2: it learns
which MAC address sits on which port and forwards Ethernet frames between
them. VMs "plug in" to a bridge with their virtual NICs.

**Why isolated:** with no physical port (`bridge-ports none`) and no IP on the
host (`inet manual`), the bridge only connects the VMs on it. Lab traffic can't
leak into the home network, and a lab mistake can't take down real access.

**Risk:** creating bridges doesn't touch `vmbr0`, but *Apply Configuration*
reloads all host networking. A bad config could take `vmbr0` down and cut off
remote access. Mitigation: backup first, read the diff, then apply.

### Where things are in the Proxmox web UI

- The **node** is the physical Proxmox server (here named `kali`). It sits
  under *Datacenter* in the left tree. It is not the same as the Kali **VM**
  (`100 (Kali-Terminal)`).
- **Node → Shell** is a root terminal on the host. Proxmox commands (`qm`,
  `pvesm`, …) go here.
- **VM → Console** is a terminal *inside* a VM. Host commands don't work there.
- Bridges are created under **Node → System → Network → Create → Linux
  Bridge**, not with *Create VM* / *Create CT* (those make machines).

### Commands and clicks

1. Backup (Node → Shell):
   ```
   cp /etc/network/interfaces /root/interfaces.bak
   ls -l /root/interfaces.bak
   ```
   `cp` prints nothing when it succeeds.
2. Node → System → Network → Create → Linux Bridge:
   - `vmbr1`, IPv4/Gateway/Bridge ports empty, Autostart on,
     comment `lab: CHR1-CHR2 /30`
   - `vmbr2`, same settings, comment `lab: CHR2-Kali LAN`
3. Review **Pending changes** before applying. Only `+` lines (additions);
   nothing touching `vmbr0`:
   ```
   +auto vmbr1
   +iface vmbr1 inet manual
   +	bridge-ports none
   +	bridge-stp off
   +	bridge-fd 0
   +#lab: CHR1-CHR2 /30
   ```
   (Same block for `vmbr2`, plus Proxmox's standard comment header.)
4. Click **Apply Configuration**.

### Verification

- Network page: `vmbr1` and `vmbr2` show **Active: Yes**; `vmbr0` still has
  its IP and gateway.
- Task log: `SRV networking - Reload` → **OK**.
- Shell: `ip -br link show type bridge` lists `vmbr1` and `vmbr2`. An empty
  bridge can show `DOWN`/`UNKNOWN` until a VM is attached; that's normal.

### Rollback

```
cp /root/interfaces.bak /etc/network/interfaces
ifreload -a
```

## Step 2: download the CHR image ⏭ next

CHR (Cloud Hosted Router) is RouterOS packaged as a VM disk image. Download it
to the host into its own folder:

```
wget -qO- https://upgrade.mikrotik.com/routeros/NEWESTa7.stable; echo
mkdir -p /root/chr && cd /root/chr
V=7.24.5
wget https://download.mikrotik.com/routeros/$V/chr-$V.img.zip
```

## Step 3: create the CHR VMs (to do)

Checked against vendor docs (see
[reference/chr-on-proxmox.md](reference/chr-on-proxmox.md)): VirtIO Block
disk, VirtIO NIC, SeaBIOS (default), 1 GB RAM.

## Step 4: add a lab NIC to Kali (to do)
