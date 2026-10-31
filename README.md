## HomeLab
 
A self-hosted Proxmox homelab I use to build real IT and infrastructure skills: virtualization, Linux administration, networking, DNS, remote access, and security hardening. I'm working on these alongside my B.S. in Cloud & Network Engineering (AWS track) at WGU.
 
Each build is a **camp**: one focused project with a clear goal, a finish line, verification steps, a troubleshooting log, and lessons learned.
 
---
 
## 🖥️ Hardware
 
| Component | Spec |
|---|---|
| Server | Dell OptiPlex 5050 SFF |
| CPU | Intel Core i5-6500 (4 cores, VT-x / VT-d) |
| RAM | 16 GB DDR4-2133 |
| NVMe | 256 GB Samsung SM951 (Proxmox + VM disks) |
| HDD | 1 TB Toshiba (ISOs, backups, templates) |
| NIC | Onboard Intel Gigabit, wired to router |
| Hypervisor | Proxmox VE 9.2 |
 
---
 
## 🏕️ Camps
 
| Camp | Project | Status | Docs |
|---|---|---|---|
| 1 | Proxmox foundation: BIOS, install, repos, storage | ✅ Complete | [camp-1-proxmox.md](camp-1-proxmox.md) |
| 2A | First Linux VM: Ubuntu Server, SSH, updates, LVM extend, snapshot | ✅ Complete | [camp-2-session-a.md](camp-2-session-a.md) |
| 2B–C | Static IP (netplan) and SSH/host hardening: key-only SSH, UFW, unattended upgrades | ✅ Complete | [camp-2-sessions-b-c.md](camp-2-sessions-b-c.md) |
| 3 | Tailscale: secure remote access to the lab | 🟡 Running · write-up in progress | — |
| 4 | AdGuard Home: network-wide DNS filtering | 🟡 Running · write-up in progress | — |
| 5 | VLAN segmentation on a Cisco managed switch | ⏭️ Next | — |
| 6 | Windows Server / Active Directory | ⏭️ Planned | — |
 
---
 
## 🧠 Lessons Learned
 
- **Verify before acting.** Check the checksum, the target disk, the repo row, and the summary screen before clicking install.
- **Confirm with evidence, not assumptions.** For example, use `uname -r` to prove a kernel update applied.
- **Plan for rollback.** Use `netplan try`, snapshots before risky changes, and a second SSH session open while hardening.
- **Don't create a bigger problem to solve a small one.** On a shared home network, that rules out a router factory reset to fix a DHCP issue.
---
 
## 🛠️ Skills Practiced
 
Proxmox VE · Virtualization · Ubuntu Server · Linux CLI · SSH (key-based auth) · UFW firewall · netplan / static IP · LVM · Snapshots & backups · Tailscale · AdGuard Home / DNS · Troubleshooting & documentation
*Built alongside a B.S. in Cloud & Network Engineering (AWS track) at WGU.*
