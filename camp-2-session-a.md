# Camp 2, Session A: First Linux VM (ubuntu-01)

- **Date:** 2026-10-07
- **Status:** ✅ Session A complete
- **Goal:** Get an Ubuntu Server VM running on Proxmox, managed over SSH from my laptop.
- **Finish line:** SSH working, system fully updated, root disk using the full virtual disk, guest agent reporting to Proxmox, clean snapshot saved.

---

## VM Specs

| Setting | Value | Why |
|---|---|---|
| OS | Ubuntu Server 26.04.1 LTS (kernel 7.0.0-38) | Current LTS = 5 years of support; most common server OS in industry and AWS |
| VM ID / Name | 100 / `ubuntu-01` | |
| CPU | 2 cores, type `host` | Passes the real CPU features through; best performance on a single node |
| RAM | 2 GiB | No desktop, so actual usage is ~550 MiB (27%) |
| Disk | 32 GB on `local-lvm` (NVMe), VirtIO SCSI single, discard + iothread + SSD emulation | Fast storage for VM disks; discard returns freed space to the thin pool |
| NIC | VirtIO on `vmbr0` | Paravirtualized network card, faster than emulating real hardware |
| ISO source | Downloaded straight to `hdd-storage` via Proxmox "Download from URL" | No laptop upload needed; SHA-256 verified automatically |
| IP | `10.0.0.104` (DHCP), moving to `10.0.0.240` static in Session B | |

## What I Did

1. **Downloaded the ISO server-side** and verified it against Ubuntu's published `SHA256SUMS`.
2. **Created and installed the VM**, with OpenSSH server selected during the install.
3. **Connected over SSH from Windows:** `ssh mike@10.0.0.104` → accepted the host fingerprint.
4. **Updated the system:** `sudo apt update && sudo apt full-upgrade -y` (21 updates → 0).
5. **Set the timezone:** `sudo timedatectl set-timezone America/Los_Angeles`.
6. **Extended the root volume while running:**
   ```bash
   sudo lvextend -r -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
   ```
   The installer only allocated ~15 GB to `/`. This grew it to ~30 GB live, with no reboot needed for the resize.
7. **Installed the QEMU guest agent:** `sudo apt install -y qemu-guest-agent`.
8. **Detached the installer ISO** (CD/DVD → no media).
9. **Took snapshot `clean-install`** as a save point before network and security changes.

## Verification

| Check | Command / Location | Result |
|---|---|---|
| Disk size | `df -h /` | 30G size, 19% used |
| Timezone | `timedatectl` | America/Los_Angeles (PDT), NTP synchronized |
| Guest agent | `systemctl status qemu-guest-agent` | active (running) |
| Agent → host | Proxmox VM Summary | IPs field shows `10.0.0.104` |
| Updates | Login banner | 0 updates pending |
| Snapshot | Proxmox task log | `TASK OK` |

## Troubleshooting Log

| Symptom | Diagnosis | Fix |
|---|---|---|
| Proxmox "Download from URL" showed 75 KB, `text/html` | Copied link pointed to Ubuntu's "thank you" web page, not the ISO | Used the direct `releases.ubuntu.com` file URL; confirmed 2.73 GiB and `application/x-iso9660-image` before downloading |
| Checksum field rejected | Pasted the whole line (hash + filename) | Pasted the hash only |
| SSH `Permission denied` twice | Fingerprint prompt and password prompt appeared, so network and SSH service were working; the problem was isolated to authentication | Password typo; re-entered carefully |
| Typed `summary` in the VM shell | Confused a Proxmox UI page with a Linux command | Cancelled the `sudo` prompt; never install suggested packages or `sudo` unknown commands |

## Concepts Learned

- **SSH host fingerprint:** the client remembers the server's identity on first connect. A future mismatch warning means the host changed or something is intercepting the connection.
- **`$` vs `#` prompt / `sudo`:** work as a normal user and elevate per command. That's least privilege.
- **LVM:** logical volumes can grow online. Same idea as expanding an AWS EBS volume.
- **Guest agent:** lets the hypervisor see inside the VM (IPs, clean shutdowns).
- **Snapshots:** thin LVM snapshots track changes only, so they're instant. They're a rollback point, **not a backup** (same physical disk).

## Next: Session B, Static IP

Move `ubuntu-01` from DHCP `.104` to static `10.0.0.240` with netplan, applied from the Proxmox console using `netplan try` (auto-rollback in 120s). After that come SSH keys, UFW, disabling password auth, and a `hardened` snapshot.
