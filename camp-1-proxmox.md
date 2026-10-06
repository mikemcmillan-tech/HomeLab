# Camp 1 — Proxmox Foundation

**Date:** 2026-10-06
**Status:** ✅ Complete
**Goal:** Turn a used Dell OptiPlex into a headless Proxmox VE hypervisor I manage from my laptop's browser.
**Finish line:** Web UI reachable on a static IP, system updated, tiered storage provisioned.

---

## Hardware

| Component | Spec |
|---|---|
| Server | Dell OptiPlex 5050 SFF |
| CPU | Intel Core i5-6500 (4 cores, VT-x / VT-d) |
| RAM | 16 GB DDR4-2133 |
| Disk 1 | 256 GB Samsung SM951 NVMe — Proxmox + VM disks |
| Disk 2 | 1 TB Toshiba DT01ACA100 HDD — ISOs, backups, templates |
| Network | Onboard Intel Gigabit (e1000e), wired to router |

## Network Plan

| Setting | Value | Why |
|---|---|---|
| Subnet | 10.0.0.0/24 | Existing home network |
| Gateway | 10.0.0.1 | ISP router |
| Proxmox IP | 10.0.0.250 (static) | Top of range, unlikely to collide with DHCP-assigned devices |
| DNS | 75.75.75.75 | ISP resolver, already proven on this network |
| Hostname | pve.home.lan | Internal-only name |

> Open issue: no DHCP reservation yet (router admin access pending). Static at .250 is a calculated risk until then.

## BIOS Changes

| Setting | Changed to | Why |
|---|---|---|
| SATA Operation | AHCI | Standard storage mode Linux expects; Dell's "RAID On" relies on a Windows-only driver |
| Virtualization (VT-x) | Enabled | Lets the CPU run VMs at full speed |
| VT for Direct I/O (VT-d) | Enabled | Allows passing hardware (e.g., iGPU) to VMs later |
| Secure Boot | Disabled | Removes a variable for the first install; revisit later |
| AC Recovery | Power On | Server comes back on its own after a power outage |

## Install & Verification

1. Downloaded the Proxmox VE 9.2 ISO and **verified its SHA256 checksum** against the published hash.
2. Flashed to USB with balenaEtcher.
3. Installed to the NVMe (ext4), timezone America/Los_Angeles, static network settings above.
4. Verified:
   - Console shows `https://10.0.0.250:8006/`
   - Web UI login works from laptop (`root`, Linux PAM)

## Repositories & Updates

- Disabled `pve-enterprise` and Ceph enterprise repos (require a paid subscription → `401` errors)
- Enabled `pve-no-subscription`
- Ran `apt update && apt full-upgrade -y`, rebooted
- Verified new kernel: `uname -r` → `7.0.14-20-pve`

## Storage

- Checked HDD health with **S.M.A.R.T.** before trusting it:
  - Reallocated_Sector_Ct: 0 · Current_Pending_Sector: 0 · Offline_Uncorrectable: 0
  - ~4,569 power-on hours, 42 °C — healthy
- Wiped old NTFS partitions, created ext4 Directory storage `hdd-storage`
- Mounted at `/mnt/pve/hdd-storage`, referenced by **UUID** (stable even if device names change)
- Tiering: NVMe = VM disks (speed) · HDD = ISOs, backups, templates (capacity)

## Troubleshooting Log

| Symptom | Cause | Fix |
|---|---|---|
| Downloaded wrong installer | Grabbed the ARM64 ISO (top of page) for an x86-64 CPU | Re-downloaded the standard build; **checksum would have caught it** |
| Etcher: "writer process ended unexpectedly" | Expired third-party antivirus (ransomware/USB protection) still hooking USB writes; two AV suites installed | Removed both, restored Microsoft Defender, re-flashed successfully |
| F12 boot menu never appeared | Fast POST / keyboard timing | Booted USB via Windows → Advanced startup → Use a device |
| Installer stalled "testing /dev/sr0" | Installer checks the DVD drive first | Waited; it found the USB on its own |
| Enterprise repo warning persisted | Disabled the wrong repo row | Re-enabled no-subscription, disabled enterprise — verify the selected row before acting |
| Router admin login failed | Admin password unknown/changed | Did **not** factory reset (would drop the whole household); used static IP outside common DHCP range instead |

## Security Notes

- Verified installer integrity with SHA256
- No ports forwarded, no DMZ — server is reachable only on the home LAN
- A reused password was flagged in a known breach → rotating it
- Credentials live in a password manager only (never on paper)

## Lessons Learned

- Verify before you act: checksum, target disk, selected row, summary screen.
- Change → verify. "I updated it" means nothing until `uname -r` proves it.
- Solve the problem without creating a bigger one (no router reset on a shared home network).

## Next: Camp 2 — First Linux VM

Ubuntu Server VM: resource sizing, install, static IP, SSH, users and permissions, package management.
