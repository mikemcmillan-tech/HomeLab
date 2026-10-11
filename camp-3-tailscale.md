# Camp 3, Session 1: Tailscale Remote Access

- **Date:** 2026-10-10
- **Status:** ✅ Session 1 complete
- **Goal:** Reach the Proxmox host from anywhere without opening any ports on the home router.
- **Finish line:** Proxmox web UI loaded on my phone over cellular (Wi-Fi off) via Tailscale.

---

## Why Tailscale (not port forwarding)

Port forwarding would expose the Proxmox login page to the public internet, where it gets constantly scanned by bots. Tailscale builds a private, encrypted WireGuard mesh between my own devices only. The router has **zero open ports**.

## Tailnet

| Device | Role | OS |
|---|---|---|
| `pve` | Proxmox host | Debian / Proxmox VE 9 |
| `mcmillan301` | Admin laptop | Windows 11 |
| `iphone171` | Phone | iOS |

Identity provider: GitHub (2FA enabled with an authenticator app).

## What I Did

1. Created a Tailscale account with GitHub SSO and enabled **GitHub 2FA**, since it's now the master key to the tailnet.
2. Installed Tailscale on the **Proxmox host** (not a VM), so remote access survives a VM crash:
   ```bash
   curl -fsSL https://tailscale.com/install.sh | sh
   tailscale up
   tailscale ip -4
   ```
3. Installed the client on my laptop and phone, signed in with the same account.
4. Disabled **key expiry on `pve` only**. Mobile devices keep expiry on in case they're lost.
5. Verified `tailscaled` starts on boot (`systemctl is-enabled tailscaled`).

## Verification

| Test | Result |
|---|---|
| Admin console | All 3 devices "Connected" |
| Phone on 5G (Wi-Fi off) → `https://<pve-tailscale-ip>:8006` | ✅ Proxmox dashboard loaded |
| Router port forwards | None |

## Troubleshooting Log

| Symptom | Diagnosis | Fix |
|---|---|---|
| Proxmox unreachable (SSH + web UI timed out) | Bottom-up: no response on LAN → **Layer 1**. Faulty passive Ethernet splitter shared with a game console | Reseated the connection; follow-up: replace with an unmanaged gigabit switch |
| Password typed at a shell prompt after an SSH timeout | Exposed in terminal history and a screenshot | Rotated root password with `passwd`, scrubbed PowerShell history file |
| Web UI `authentication failure (401)` | Console login worked → the password was valid; the problem was the login form (realm / typo) | Use `root` + **Linux PAM** realm |
| `pve` showed "last seen" instead of Connected; Tailscale IP timed out while LAN ping worked | Isolated to Tailscale on the host. `tailscale up` was never fully completed/approved | Re-ran `tailscale up`, approved the device |
| Phone login: `Error 401 bad tailscale-authstate2 cookie` | Auth flow lost its session cookie mid-login | Closed the app, retried sign-in in one browser session |
| Safari "Connection Is Not Private" | Proxmox uses a self-signed certificate; the connection is still encrypted (HTTPS + WireGuard) | Proceeded, only because it's my own server |

## Concepts Learned

- **Overlay network / mesh VPN:** devices get stable `100.x.y.z` addresses on a private network that rides over any internet connection.
- **WireGuard:** modern, fast VPN protocol; Tailscale handles key exchange and NAT traversal.
- **Zero-trust access:** being on the tailnet is the gate, not being on the home LAN.
- **Identity is the perimeter:** whoever controls the SSO account controls the network, so 2FA is non-negotiable.
- **Bottom-up troubleshooting:** physical → network → service → auth. Two "timeouts" today had two different causes.

## Next: Session 2, Subnet Router

Enable IP forwarding on `pve` and advertise `10.0.0.0/24` so the whole lab (e.g., `ubuntu-01`) is reachable remotely without installing Tailscale on every VM.
