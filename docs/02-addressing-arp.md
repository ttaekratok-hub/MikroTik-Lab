# Milestone 2: addressing, ARP, L2 vs L3

Goal: give every port an IP address, watch ARP connect IP to MAC, and see
why devices on different subnets can't talk without routes.

## Addressing plan

| Device | Port | Address | Network | Faces |
|---|---|---|---|---|
| CHR1 | `ether1` | `10.0.12.1/30` | `10.0.12.0/30` | CHR2 |
| CHR2 | `ether1` | `10.0.12.2/30` | `10.0.12.0/30` | CHR1 |
| CHR2 | `ether2` | `192.168.20.1/24` | `192.168.20.0/24` | Kali LAN |
| Kali | `eth1` | `192.168.20.10/24` | `192.168.20.0/24` | CHR2 |

- **/30** = 4 addresses: `.0` network, `.1` and `.2` hosts, `.3` broadcast.
  A link between exactly two routers needs exactly two addresses, so ISPs
  use /30 (or /31) on point-to-point links.
- **/24** = 256 addresses, 254 usable. Used for a LAN with many devices. By
  convention the router takes `.1`, the LAN's default gateway.

## L2 vs L3: when were CHR1 and CHR2 connected?

| When | What | Layer | Proof |
|---|---|---|---|
| Milestone 1 (`qm set … bridge=vmbr1`) | Both ports plugged into switch `vmbr1` | L2: frames can flow | `R` flag in `/interface print` |
| Milestone 2 (`/ip address add …`) | Both ends in the same subnet | L3: IP can flow | ping works, ARP entry appears |

Sharing a bridge *is* the cable. Before the IPs, the cable was plugged in but
the routers had no addresses.

## Step 1: IPs on the CHR1–CHR2 link

```
/ip address add address=10.0.12.1/30 interface=ether1    # CHR1
/ip address add address=10.0.12.2/30 interface=ether1    # CHR2
/ip address print
```

Both show NETWORK `10.0.12.0`: same subnet, so they are directly connected.

Mistakes I made, and what the errors meant:

- `/ip add …` → `syntax error`. `/ip` is a menu of menus; the full path is
  `/ip address add`.
- `10.0.12.1.30/30` → `invalid value`. Five numbers; IPv4 has four. The
  `/30` already gives the mask.

## Step 2: ping and ARP

```
/ping 10.0.12.2 count=4      # CHR1
/ip arp print
```

- 4/4 received, ~0.8 ms, TTL 64 (no router in between).
- ARP table: `10.0.12.2 → BC:24:11:04:32:23`, which is CHR2's `ether1` MAC.
  Flags `D` (dynamic, learned) and `C` (complete).

Before the first ping, CHR1 broadcasts "who has 10.0.12.2?" and CHR2 answers
with its MAC. **IP decides who to send to, ARP finds which MAC, the switch
delivers by MAC.**

## Step 3: the LAN side

```
/ip address add address=192.168.20.1/24 interface=ether2    # CHR2
```

CHR2 now has one address on each network: that's what makes it a router.

## Step 4–5: Kali's lab NIC

Match the NIC by MAC first: `ip -br link` showed `eth1` = `…44:7e:2f` =
Proxmox `net1` on `vmbr2`. Then a saved NetworkManager connection with a
static address and **no gateway** (internet stays on `eth0`):

```
sudo nmcli con add type ethernet ifname eth1 con-name lab
sudo nmcli con mod lab ipv4.method manual ipv4.addresses 192.168.20.10/24
sudo nmcli con up lab
ping -c 4 192.168.20.1
```

**Side problem:** after the second NIC appeared, `eth0` came up
*disconnected*: its profile `Wired connection 1` wasn't tied to a device.
Fix:

```
sudo nmcli dev connect eth0
sudo nmcli con mod "Wired connection 1" connection.interface-name eth0
```

## Step 6–8: why different networks can't talk without routes

Networks are streets; the network part of the address is the street name.
A device can reach any house on its own street directly. For another street
it needs a **route**: a signpost saying "to reach street X, go through door Y".

| From Kali | Result | Why |
|---|---|---|
| ping `10.0.12.2`, no default route | `Network is unreachable` | No matching route: the packet never left Kali |
| ping `10.0.12.2`, default via home router | 100% loss (timeout) | Default route matched; the home router sent it toward the internet, where it died |
| add `10.0.0.0/8 via 192.168.20.1`, ping `10.0.12.2` | ✅ works | `/8` beats `/0` (longest prefix match), CHR2 owns the address |
| ping `10.0.12.1` (CHR1) | 100% loss | Request reaches CHR1, but CHR1 has no route back to `192.168.20.0/24` |

Lab route on Kali (saved in the `lab` connection):

```
sudo nmcli con mod lab +ipv4.routes "10.0.0.0/8 192.168.20.1"
sudo nmcli con up lab
```

Forgetting `lab` gives `Error: unknown connection '+ipv4.routes'`: nmcli
takes the word after `mod` as the connection name.

Lessons:

- **Different errors mean different failure points.** "Unreachable" = no
  route on this device. Timeout = the packet left, but something along the
  way dropped it or couldn't answer.
- **Longest prefix match:** when several routes match, the most specific
  (biggest `/`) wins.
- **Hop-by-hop:** a route only points to the next hop; each router decides
  with its own table.
- **Routing must work in both directions.** A missing return route looks
  exactly like a timeout from the sender's side.

Proof on CHR1:

```
/ip route print
```

```
DAc 10.0.12.0/30  ether1  main  0
```

Only one route, its own connected subnet (`D` dynamic, `A` active,
`c` connected, distance 0). Nothing for `192.168.20.0/24`, no default route.
Fixed in milestone 3.

## Milestone 2 complete ✅
