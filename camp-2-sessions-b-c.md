# Camp 2, Sessions B & C: Static IP + Hardening (ubuntu-01)

- **Dates:** 2026-10-08 → 2026-10-09
- **Status:** ✅ Camp 2 complete
- **Goal:** Give `ubuntu-01` a permanent address and lock it down like a production server.
- **Finish line:** Static IP that survives reboots, key-only SSH, host firewall on, auto security updates, `hardened` snapshot.

---

## Session B: Static IP

**Why:** DHCP addresses can change. With no router admin access for a DHCP reservation, I pinned the address inside Ubuntu instead.

| Setting | Value |
|---|---|
| IPv4 | `10.0.0.240/24` (static, was DHCP `.104`) |
| Gateway | `10.0.0.1` |
| DNS | `75.75.75.75` |
| IPv6 | Left on automatic (`dhcp6: true`) |
| Config file | `/etc/netplan/00-installer-config.yaml` |

**Process:**
1. Read the existing netplan config first. It was `00-installer-config.yaml` (written by the Ubuntu installer), not the `50-cloud-init.yaml` I expected.
2. Backed it up to `~/netplan-backup.yaml`.
3. Wrote the static config (kept the `match: macaddress` + `set-name` block).
4. Applied it from the **Proxmox console** (not SSH) with `sudo netplan try`, which auto-reverts after 120s unless confirmed.
5. Verified from my **laptop** while the timer ran: `ping 10.0.0.240` + `ssh mike@10.0.0.240`.
6. Rebooted and confirmed the IP persisted (`valid_lft forever`).

## Session C: Hardening

| Control | Implementation | Verification |
|---|---|---|
| Key-based SSH | ed25519 key pair generated on Windows; public key → `~/.ssh/authorized_keys` | `ssh -o PasswordAuthentication=no` logs in with key passphrase |
| Host firewall | `ufw` default deny incoming, allow outgoing, allow OpenSSH (22/tcp) | `sudo ufw status verbose` → active |
| Password login disabled | `/etc/ssh/sshd_config.d/00-hardening.conf`: `PasswordAuthentication no`, `KbdInteractiveAuthentication no`, `PermitRootLogin no` | `sudo sshd -T` shows `no`; `ssh -o PubkeyAuthentication=no` → **`Permission denied (publickey)`** |
| Auto security updates | `unattended-upgrades` (preinstalled, enabled) | `systemctl status` → active |

**Safety order I followed:** test key login → allow SSH in firewall → enable firewall → confirm new session works → disable passwords (with a second session open as a lifeline) → `sshd -t` config test before restart → verify from a fresh session.

## Snapshots

| Name | State |
|---|---|
| `clean-install` | Updated, 30G root, guest agent |
| `static-ip` | `.240` static networking |
| `hardened` | Keys, firewall, no password auth |

## Troubleshooting Log

| Symptom | Cause | Fix |
|---|---|---|
| `cp: cannot stat .../50-cloud-init.yaml` | Assumed filename; actual file was `00-installer-config.yaml` | Read before editing: `ls /etc/netplan/` |
| Couldn't paste in Proxmox console | noVNC console doesn't support clipboard | Typed short commands; verified from PowerShell instead |
| `ping-c: command not found` | Missing space between command and option | `ping -c 3 ...` |
| `ping google.com` succeeded over IPv6 | Linux prefers IPv6 when available, so the test didn't prove the IPv4 path | `ping -4 -c 3 google.com` |
| `ip: not recognized` in PowerShell | Reboot dropped the SSH session back to Windows | Reconnect first; always check the prompt |
| Pasted command into `ssh-keygen`'s filename prompt | Pasted before answering the interactive prompt | Ctrl+C, re-ran, pressed Enter at each prompt |
| `ufw status`: "You need to be root" | Missing `sudo` | `sudo ufw status verbose` |
| SSH test ran from the server to itself | Ran a laptop command inside the SSH session | Check the prompt: `PS C:\` = laptop, `mike@ubuntu-01` = server |
| systemd "unit file changed on disk" warning | Earlier update replaced the ssh unit file | `sudo systemctl daemon-reload` |

## Concepts Learned

- **netplan:** Ubuntu's declarative network config (YAML). `netplan try` = change with automatic rollback.
- **Public/private keys:** the private key never leaves the laptop; the server only holds the public key and sends a challenge only the private key can answer.
- **SSH login vs `sudo`:** keys replace the *login* password; `sudo` still requires the account password (a second factor for admin actions).
- **Default-deny firewall:** block everything, allow only what's needed. Same model as AWS security groups.
- **sshd drop-in precedence:** SSH uses the *first* value it reads, so `00-` prefix ensures my settings win.
- **Defense in depth:** keys + no passwords + no root login + firewall + auto-patching.

## Next: Camp 3, Tailscale

Install Tailscale on the Proxmox host for secure remote access from anywhere, with zero open ports on the router.
