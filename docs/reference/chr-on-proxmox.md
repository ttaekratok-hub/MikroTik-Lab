# Reference: running MikroTik CHR on Proxmox

Facts checked against vendor docs on 2026-10-07. Verify again before relying
on version numbers.

## Image

- Download `chr-<version>.img.zip` (the x86 raw image, not `-arm64`) from
  mikrotik.com/download/chr.
- Current version: `wget -qO- https://upgrade.mikrotik.com/routeros/NEWESTa7.stable`
  (7.24.5 stable and 7.23.7 long-term at the time of writing).
- `unzip` isn't installed on stock Proxmox. `busybox unzip` or
  `python3 -m zipfile -e <zip> .` work.

## VM settings

| Setting | Use | Why |
|---|---|---|
| Disk bus | VirtIO Block (`virtio0`) | Documented by MikroTik. VirtIO SCSI only has community reports. |
| NIC model | `virtio` | Supported, and needed for Fast Path. Proxmox defaults to e1000, so set it explicitly. |
| Firmware | SeaBIOS (default), no EFI disk | The stock image is reported not to boot under OVMF/UEFI. |
| RAM | 1024 MiB | MikroTik's recommended minimum. Proxmox's CLI default is 512. |
| vCPU | 1 | Enough for a lab |
| Boot | `--boot order=virtio0` | `bootdisk` is deprecated |
| Import | `qm disk import <vmid> <img> <storage>` | `qm importdisk` is the old alias |

## First login

- User `admin`, empty password. Set a strong password at the prompt.
- No firewall rules by default, and many services enabled (telnet, ftp, www,
  ssh, api, winbox). Lock them down even in a lab, as an ISP would.

## Licensing

| License | Speed limit per interface | Price (perpetual) |
|---|---|---|
| Free | 1 Mbit/s upload | $0 |
| P1 | 1 Gbit/s | $45 |
| P10 | 10 Gbit/s | $95 |
| P-Unlimited | none | $250 |

A 60-day trial needs a mikrotik.com account and internet access from the CHR
(`/system license renew`). The free license is enough for this lab.

## Sources

- help.mikrotik.com: Cloud Hosted Router (CHR), and CHR licensing
- mikrotik.com/download/chr
- Proxmox VE admin guide, `qm(1)` and `qm.conf(5)`
