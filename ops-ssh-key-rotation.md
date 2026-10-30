# Ops: SSH Key Rotation + Device Retirement

- **Date:** 2026-10-30
- **Status:** ✅ Complete
- **Goal:** Give my new laptop (`mike-yoga`) its own SSH key, revoke the retired laptop's key on every host, and remove the old device from the tailnet.
- **Why:** My new laptop was using a private key copied from the old one. One device = one key, so a lost or retired device can be revoked without touching the others.

---

## Plan: add → test → remove → verify

Never remove a lock before the new key is proven to work.

| Step | Action |
|---|---|
| 1 | Generate a new ed25519 key on the new laptop (separate filename, so nothing gets overwritten) |
| 2 | Append its public key to each host using the existing access |
| 3 | Prove the new key works on its own (`IdentitiesOnly=yes`) |
| 4 | Back up `authorized_keys`, remove the old key's line, verify the new key still works |
| 5 | Prove the old key is denied |
| 6 | Remove the old device from Tailscale and delete the old key files |

## Commands

New key + SSH config shortcuts:
```powershell
ssh-keygen -t ed25519 -C "mike-yoga" -f "$HOME\.ssh\id_ed25519_yoga"
Get-Content "$HOME\.ssh\id_ed25519_yoga.pub" | ssh root@10.0.0.250 "cat >> /root/.ssh/authorized_keys"
ssh -i "$HOME\.ssh\id_ed25519_yoga" -o IdentitiesOnly=yes root@10.0.0.250 hostname
```

`~/.ssh/config` (so `ssh pve` and `ssh ubuntu-01` just work):
```
Host pve
    HostName 10.0.0.250
    User root
    IdentityFile ~/.ssh/id_ed25519_yoga
    IdentitiesOnly yes

Host ubuntu-01
    HostName 10.0.0.240
    User mike
    IdentityFile ~/.ssh/id_ed25519_yoga
    IdentitiesOnly yes
```

Removing the old key (Proxmox host):
```bash
cp /etc/pve/priv/authorized_keys /root/authorized_keys.bak
tr -d '\r' < /root/authorized_keys.bak | grep -v 'mike-laptop' > /root/ak.new
cat -n /root/ak.new                              # verify before writing
cat /root/ak.new > /root/.ssh/authorized_keys    # write THROUGH the symlink
```

## Verification

| Test | pve | ubuntu-01 |
|---|---|---|
| New key (`ssh pve` / `ssh ubuntu-01`) | ✅ `pve` | ✅ `ubuntu-01` |
| Old key (`-i id_ed25519 -o IdentitiesOnly=yes`) | ✅ `Permission denied (publickey)` | ✅ `Permission denied (publickey)` |
| Proxmox's own `root@pve` key | ✅ Untouched | n/a |
| Old laptop in Tailscale | Removed from admin console | |

## Troubleshooting Log

| Symptom | Diagnosis | Fix |
|---|---|---|
| Notepad wanted to create `config.txt` | SSH only reads a file named exactly `config` | Wrote the file from PowerShell with `Set-Content` |
| Old key still accepted after "removing" it | `ak.new` was **921 bytes, same as the backup**, so the filter removed nothing. Keys added from PowerShell ended in a hidden `\r` (Windows CRLF), so the pattern `' mike-laptop$'` never matched | `tr -d '\r'` first, match without the `$` anchor. File shrank to 826 bytes; old key denied |
| Risk: `authorized_keys` on Proxmox is a **symlink** to `/etc/pve/priv/authorized_keys` | `sed -i` would replace the link with a plain file and break Proxmox's key management | Wrote through the link with `cat > ...`; confirmed with `ls -l` that the link survived |
| New key "denied" on ubuntu-01 right after the edit | One passphrase prompt then denied, while the same key had worked seconds earlier, pointing to a passphrase typo, not the server | Retried carefully → worked. Lifeline kept ready: Proxmox console login + `~/ak.bak` |

## Concepts

- **Per-device keys:** revoke one device without disrupting others (same idea as per-user IAM credentials in AWS).
- **Public vs. private key:** the server only holds public keys; deleting a line from `authorized_keys` revokes that device.
- **CRLF vs. LF:** Windows line endings carry an invisible `\r` that can break Linux text matching.
- **Symlinks:** edit the target, not the link, when a tool manages the real file.
- **Verify by measurement:** the unchanged file size proved the first removal failed before any test did.

**Cert mapping:** Security+ (key management, credential revocation, device decommissioning) · Linux+ (`authorized_keys`, symlinks, `grep`/`tr`, file redirection) · A+ (troubleshooting methodology) · AWS (per-identity credentials, least privilege)
