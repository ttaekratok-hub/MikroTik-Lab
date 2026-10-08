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

- The **node** is the physical Proxmox server (here named `themonitor`, formerly `kali`). It sits
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

## Step 2: download and unzip the CHR image ✅

CHR (Cloud Hosted Router) is RouterOS, MikroTik's router OS, packaged as a VM
disk image for normal x86 servers. Same commands, OSPF, BGP and firewall as a
physical MikroTik router. The `.img` is a byte-for-byte copy of a router's
disk; Proxmox imports it as the VM's disk.

```
wget -qO- https://upgrade.mikrotik.com/routeros/NEWESTa7.stable; echo
mkdir -p /root/chr && cd /root/chr
V=7.24.5
wget https://download.mikrotik.com/routeros/$V/chr-$V.img.zip
busybox unzip chr-$V.img.zip
ls -lh
```

Stock Proxmox has no `unzip`; `busybox unzip` works. Result:
`chr-7.24.5.img`, 128M.

Before creating the routers I stopped Kali (VM 100) and an unused dev VM to
free RAM: the host has ~7 GB and each CHR gets 1 GB.

## Step 3: create the CHR VMs ✅

Settings checked against vendor docs (see
[reference/chr-on-proxmox.md](reference/chr-on-proxmox.md)): VirtIO Block
disk, VirtIO NIC, SeaBIOS (default), 1 GB RAM. Built in small commands
because long lines get cut off in the web console.

CHR1 (one NIC, toward CHR2):

```
qm create 101 --name CHR1 --memory 1024
qm set 101 --net0 virtio,bridge=vmbr1
qm disk import 101 chr-7.24.5.img local-lvm
qm set 101 --virtio0 local-lvm:vm-101-disk-0
qm set 101 --boot order=virtio0
qm config 101
```

`qm disk import` attaches the new disk as `unused0`; `--virtio0` attaches it
for real, after which `unused0` disappears.

CHR2 (two NICs, the router in the middle): same commands with `102`, plus
`qm set 102 --net1 virtio,bridge=vmbr2`.

### Verification

`qm config` shows `memory: 1024`, `boot: order=virtio0`,
`virtio0: local-lvm:vm-10X-disk-0,size=128M` and the NICs:

| VM | Proxmox NIC | RouterOS name | Bridge | Faces |
|---|---|---|---|---|
| CHR1 (101) | `net0` | `ether1` | `vmbr1` | CHR2 |
| CHR2 (102) | `net0` | `ether1` | `vmbr1` | CHR1 |
| CHR2 (102) | `net1` | `ether2` | `vmbr2` | Kali |

RouterOS numbers ports from 1, so Proxmox `net0` = `ether1`, `net1` = `ether2`.

### Rollback

```
qm destroy 101
qm destroy 102
```

## Step 4: add a lab NIC to Kali ✅

Kali is the "customer PC" on CHR2's LAN. It gets a **second** NIC on `vmbr2`;
`net0` stays on `vmbr0` so Kali keeps its home-network and internet access.
Added while Kali was stopped.

```
qm set 100 --net1 virtio,bridge=vmbr2
qm config 100 | grep net
```

Result: `net0` unchanged (`bridge=vmbr0,firewall=1`), new
`net1: virtio=...,bridge=vmbr2`.

Rollback: `qm set 100 --delete net1`

## Mental model

Think of it as hardware on a desk: each bridge is an unplugged network switch,
each `netN` is a port on a machine with a cable to one switch.

```
CHR1 ether1 --[switch vmbr1]-- ether1 CHR2 ether2 --[switch vmbr2]-- Kali net1
```

## Step 5: first boot, admin password, identity ✅

A fresh CHR has user `admin` with **no password** and all management services
on. First job, as at an ISP: set a password and give the router a clear name.

```
qm start 101          # Node → Shell; then VM 101 → Console
```

Log in as `admin` with an empty password, answer `n` to the license
question. At `new password>` pressing Enter skips the step. If that happens,
set it afterwards:

```
/password
/interface print
/system identity set name=CHR1
```

Same for CHR2 (`qm start 102`, identity `CHR2`).

RouterOS is not Linux: `ls` gives `bad command name`. Commands are menus
starting with `/` (e.g. `/interface print`). Tab completes, `?` lists options,
F1 is help.

### Verification

`/interface print` lists the ports with flag `R` (running = link up). The
MAC addresses match `qm config`, which proves the Proxmox-to-RouterOS mapping:

| Router | RouterOS port | Matches Proxmox |
|---|---|---|
| CHR1 | `ether1` | `net0` (`vmbr1`) |
| CHR2 | `ether1` | `net0` (`vmbr1`) |
| CHR2 | `ether2` | `net1` (`vmbr2`) |

`lo` is the loopback: a virtual interface inside the router with no cable.
The prompt changes to `[admin@CHR1] >` / `[admin@CHR2] >`. Naming routers
matters: with two consoles open it's easy to type into the wrong one.

Passwords are kept in a password manager, never in this repo.

## Milestone 1 complete ✅
