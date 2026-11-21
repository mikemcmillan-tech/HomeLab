# Camp 6: Proxmox Backup Server

- **Dates:** 2026-11-19 → 2026-11-20
- **Status:** ✅ Complete
- **Goal:** Get backups off the machine they protect. A dedicated backup server receives nightly backups of every VM and container.
- **Finish line:** Automated nightly backups to a second box, server-side retention, and a **successful restore test**.

---

## Why

Before this camp, my container backups lived on the same Proxmox host's HDD. If that host died, the backups died with it. A backup on the same box is a copy, not a backup.

## Hardware

| Item | Value |
|---|---|
| Machine | Dell OptiPlex 5050 SFF (#2): i5-7500, 16 GB RAM |
| OS disk | 128 GB SanDisk X600 M.2 (`/dev/sdb`) |
| Backup disk | 256 GB Intel SSD (`/dev/sda`), S.M.A.R.T. passed |
| IP | `10.0.0.251` (static), web UI `https://10.0.0.251:8007` |
| Power | UPS battery outlet |

## Session 0: Network Prep

Replaced a passive Ethernet splitter (the cause of an earlier outage, and limited to 100 Mbps) with a **Netgear GS108 unmanaged gigabit switch**, powered from the UPS.

```
Wall jack → GS108 → pve (.250), PBS (.251), PS5, Z230 (later)
```

- Verified pve came back behind the switch (`ping 10.0.0.250`, 0% loss).
- Verified `10.0.0.251` was free before assigning it. "Destination host unreachable" from my own PC means nobody answered ARP, so the address is unused.

## Session 1: Install PBS

1. Downloaded the PBS 4.2 ISO and **verified the SHA-256** with PowerShell:
   ```powershell
   (Get-FileHash .\proxmox-backup-server_4.2-1.iso -Algorithm SHA256).Hash -eq $expected   # True
   ```
2. Flashed to USB with balenaEtcher and booted through Windows Advanced Startup (F2/F12 timing didn't catch).
3. **Caught the installer defaulting to the wrong disk.** It pre-selected the 238 GB Intel SSD. I switched it to the 119 GB SanDisk so the large drive stays free for backups.
4. Hostname `pbs.home.lan`, static `10.0.0.251/24`, gateway `10.0.0.1`, DNS `75.75.75.75`.
5. Gave PBS its **own unique root password**, not reused from pve. Backups are the last line of defense.
6. Disabled the enterprise repo, added **pbs-no-subscription**, and updated.

## Session 2: Datastore, Least-Privilege Access, Automation

### Datastore
Wiped the old Windows partitions from `/dev/sda` (the whole disk, not just one partition) and created an ext4 directory datastore named **`backups`**.

### Least-privilege access
pve does **not** log in to PBS as root. If pve were compromised, a root credential could delete every backup.

| Object | Value |
|---|---|
| User | `pve-backup@pbs` |
| API token | `pve-backup@pbs!pve` (secret stored in Bitwarden) |
| Permission | `DatastoreBackup` on `/datastore/backups` only, for both user and token |

A token can never exceed its user's permissions, so both need the role.

### Connect pve to PBS
pve → Datacenter → Storage → Add → Proxmox Backup Server, using the token and PBS's **TLS fingerprint**. The fingerprint pins PBS's identity so pve can't be tricked into sending backups to an impostor.

### Nightly job
| Setting | Value |
|---|---|
| Schedule | Daily at 02:00 |
| Selection | All guests |
| Mode | Snapshot |
| Storage | `pbs` |

### Retention (server-side)
Configured on **PBS**, not pve: keep 7 daily, 4 weekly, 3 monthly. Since pruning happens on the backup server, a compromised pve can't wipe history. Garbage collection reclaims the space.

## Verification

| Check | Result |
|---|---|
| First backup run | VM 100, CT 101, CT 102 backed up in about 77 seconds total |
| Backups visible on PBS | `vm/100`, `ct/101`, `ct/102` in the `backups` datastore |
| **Restore test** | Restored CT 102 to a new ID (902), **not started** to avoid an IP conflict, 982 MiB in 9 s, `TASK OK`. Then destroyed 902; production CT 102 was untouched |

> A backup you've never restored is a theory. This one is proven.

## Troubleshooting Log

| Symptom | Diagnosis | Fix |
|---|---|---|
| F2 didn't open the BIOS | Windows Fast Startup / keyboard timing | Booted the USB through Windows Advanced Startup |
| Installer pre-selected the 238 GB SSD | Default picks the first disk | Selected `/dev/sdb` (119 GB) and confirmed on the Summary screen |
| Pasted the SHA-256 hash alone in PowerShell → "not recognized" | A hash is a string, not a command | Put it in a variable and compared with `-eq` → `True` |
| Repo refresh showed only Debian sources | Enterprise disabled but no-subscription never added | Added the No-Subscription repo; confirmed "You get updates" |
| PBS web login failed | Bitwarden autofilled the **API token**, which can't log in to the UI by design, and the stored root password wasn't the one set at install | Logged in as `root` / Linux PAM with the correct password; saved it to Bitwarden |
| `ssh root@10.0.0.250` → permission denied | Raw IP doesn't trigger my per-host key in `~/.ssh/config` | Use the `ssh pve` alias |

## Concepts Learned

- **3-2-1 backups:** 3 copies, 2 media, 1 offsite. This camp covers "separate machine." Offsite (AWS) comes later.
- **Least privilege + API tokens:** scoped, revocable credentials for automation, separate from interactive logins.
- **TLS fingerprint pinning:** trust a specific server, not just any certificate.
- **Server-side retention:** the client can't delete history. That's ransomware resistance.
- **Restore testing:** the only way to know a backup works.
- **Credential management:** set up Bitwarden with MFA for all lab credentials.

## Camp 6 Summary

- **Cert mapping:**
  - **A+:** BIOS/boot options, disk identification, backup concepts
  - **Network+:** static addressing, switching, ARP behavior ("destination host unreachable")
  - **Linux+:** storage/partitions, filesystems, repositories, users and permissions
  - **Security+:** least privilege, API tokens, credential management, backup integrity, ransomware resilience
  - **AWS:** IAM-style scoped credentials, backup and retention policies (AWS Backup / S3 lifecycle concepts)
- **Resume bullet:** Deployed Proxmox Backup Server on dedicated hardware with automated nightly backups of all VMs and containers, server-side retention policies, and least-privilege API token authentication; validated recoverability with documented restore testing.

## Next

- UPS auto-shutdown (pve and PBS shut down cleanly on low battery)
- Offsite copy to AWS (completes 3-2-1)
