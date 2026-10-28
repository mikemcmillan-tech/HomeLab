# Ops Log — Oct 28, 2026: Host Recovery, New Admin Workstation, Maintenance

**Scope:** Proxmox host `pve`, VM `ubuntu-01`, Tailscale
**Outcome:** Host recovered, new workstation onboarded, VM credentials recovered, VM fully patched

---

## 1. Incident: Proxmox Host Unreachable

### Symptom
From the new laptop, `https://10.0.0.250:8006` timed out (`ERR_CONNECTION_TIMED_OUT`).

### Diagnosis (bottom-up)

| Step | Command | Finding |
|---|---|---|
| Is the client on the right network? | `ipconfig` | Laptop at `10.0.0.207/24`, gateway `10.0.0.1`: correct subnet |
| Can the client reach the host? | `ping 10.0.0.250` | `Reply from 10.0.0.207: Destination host unreachable` |
| Read the reply carefully | | The replies came from the **laptop itself**, not the server. "Received = 4" was really four error messages. Nothing on the LAN answered |
| Check the host directly | Console at the OptiPlex | Server powered on, showing the login prompt |
| After login | `ip -br a` | `nic0` UP, `vmbr0` UP at `10.0.0.250/24` |
| Host to router | `ping -c 4 10.0.0.1` | 4/4 replies, under 1 ms |

### Resolution
The host was power-cycled. The Proxmox task log shows **"Bulk start VMs and Containers" at 09:23**, which confirms a fresh boot. Access was restored right after.

### Root cause: undetermined
- The previous boot ran from Oct 20 until this morning; the last recorded activity was a scheduled task at 03:45.
- `journalctl -b -1 -p err` showed only normal startup messages (SGX disabled, TPM probe, blkmapd), **with no errors before the hang**.
- **Follow-up:** check `journalctl -b -1 -n 30` for the last messages before the restart, and watch for a repeat. If it happens again, check for network-card hangs (`dmesg | grep -i e1000`), a known issue on OptiPlex hardware running Proxmox.

> Correction: I first assumed the router had lost track of the server's address. The task-log timestamps showed the host had actually rebooted, so I updated the root cause.

---

## 2. New Admin Workstation Onboarding

- Replaced the daily laptop (see the `aplus-labs` migration write-up). The new device is `mike-yoga`.
- **SSH key migrated** by restoring `~/.ssh` from backup. Key-only login to `ubuntu-01` confirmed.
- **Stale documentation found:** a note listed the VM at `.104`, its old DHCP address. It's actually at the static **`10.0.0.240`** set in Camp 2. Fixed the note.
- **Tailscale:** joined `mike-yoga` to the tailnet using GitHub SSO (with 2FA). The tailnet now has 4 devices.
- **Offboarding plan:** remove the old laptop from the tailnet when it's wiped.

---

## 3. Credential Recovery: Forgotten VM `sudo` Password

**Symptom:** `sudo` rejected the password repeatedly. SSH still worked because it uses keys, not the password.

**Fix:** reset the password from the hypervisor through the QEMU guest agent. No reboot or recovery mode needed:
1. Took a snapshot: `pre-password-reset`
2. On the Proxmox host shell (`root@pve`):
   ```bash
   qm guest passwd 100 mike
   ```
3. Verified from SSH:
   ```bash
   sudo -v
   ```

**Lesson:** this lab has three separate secrets: the Proxmox root password, the Ubuntu user password, and the SSH key passphrase. All three now go in a password manager, not on sticky notes.

---

## 4. Maintenance

```bash
sudo apt update && sudo apt upgrade -y
sudo reboot
uname -r
```
- 22 package updates applied.
- Kernel upgraded from `7.0.0-38-generic` to **`7.0.0-42-generic`**. A reboot was required to load it.

---

## Lessons Learned

- **Read who sent the ping reply.** "Destination host unreachable" coming from your own IP means nothing on the network answered.
- **SSH errors tell you where the problem is:** *timed out* means a network or firewall problem; *refused* means the SSH service isn't running; *permission denied* means bad credentials or keys.
- **Timestamps beat assumptions.** The task log disproved my first theory about the outage.
- **Document IP addresses, not passwords.** An outdated IP note cost time. Keep the IP plan in the repo and secrets in a password manager.
- **The guest agent pays off.** Installing it in Camp 2 made today's password recovery a single command.

## Open Items

- [ ] Find what caused the hang (`journalctl -b -1 -n 30`)
- [ ] Remote-access test over a phone hotspot (Proxmox UI and SSH through the subnet router)
- [ ] Move all three lab credentials into a password manager; get rid of the sticky note
- [ ] Remove the old laptop from the tailnet once it's wiped
- [ ] Proxmox email alerts, so an outage gets noticed instead of discovered

## Cert Mapping

- **Network+:** troubleshooting methodology, ICMP/ARP, subnetting, gateway
- **Linux+:** `ip`, `journalctl`, `apt`, kernel updates, `sudo`, password management
- **Security+:** credential management, least privilege, device offboarding
- **A+ Core 2:** troubleshooting process, documentation
