# Camp 4: Jellyfin Media Server

- **Dates:** 2026-10-19 → 2026-10-20 (Sessions 1–2)
- **Status:** ⏳ In progress. Sessions 1–2 done; Session 3 (hardware transcoding + clients) next
- **Goal:** Self-hosted media server in a lightweight LXC container, using the host's Intel iGPU for transcoding.
- **Finish line (Sessions 1–2):** Jellyfin installed, libraries mapped to HDD storage, survives a full host reboot.

---

## Architecture

| Item | Value |
|---|---|
| Container | CT 101 `jellyfin`, **unprivileged** Debian 13 (trixie) LXC |
| IP | `10.0.0.241/24` (static), gateway `10.0.0.1`, DNS `75.75.75.75` |
| Media storage | Host `/mnt/pve/hdd-storage/media` → bind mount → container `/media` |
| Libraries | `/media/movies` (Movies), `/media/shows` (Shows), `/media/fight-film` (Home Videos) |
| GPU | `/dev/dri/renderD128` passed through, `gid=992` (render) |
| Web UI | `http://10.0.0.241:8096` (LAN + Tailscale subnet route). **No port forwarding** |
| Version | Jellyfin 12.2 |

**Why LXC instead of a VM:** a container shares the host kernel, so it uses a fraction of the RAM and boots in seconds. And because the iGPU is already on the host, passing a device node into a container is simpler than full PCI passthrough to a VM.

**Why unprivileged:** root inside the container maps to an unprivileged user on the host. If Jellyfin were ever compromised, the attacker would not be root on Proxmox.

## Session 1: Container + Storage + GPU

1. Created CT 101 from the Debian 13 template, unprivileged, static IP `.241`.
2. Created media folders on the HDD and bind-mounted them in:
   ```bash
   mkdir -p /mnt/pve/hdd-storage/media/{movies,shows,fight-film}
   pct set 101 -mp0 /mnt/pve/hdd-storage/media,mp=/media
   ```
3. Passed through the iGPU render node with the host's `render` group ID (`992`).
4. Tried to snapshot → **blocked** (Proxmox can't snapshot a container with a bind mount). Took a `vzdump` backup instead.

## Session 2: Install + Configure

1. Updated the container and installed prerequisites (`curl`, `gnupg`), set timezone.
2. **Verified before running:** downloaded the official installer, checked it against Jellyfin's published SHA-256 (`install-debuntu.sh: OK`), and skimmed it with `grep`. It adds Jellyfin's signing key and a repo entry with `Signed-By`, then runs `apt install jellyfin`.
3. Ran the installer, added the service account to the GPU groups, restarted:
   ```bash
   bash install-debuntu.sh
   usermod -aG render,video jellyfin
   systemctl restart jellyfin
   ```
4. Setup wizard: server name `jellyfin`, admin user `mike`, three libraries.
   - **Remote connections:** allowed (needed for Tailscale clients)
   - **Automatic port mapping (UPnP):** **off**. It would ask the router to open a port to the internet.
5. Renamed the admin account from the default-looking `jellyfin` to `mike` (predictable admin names are the first ones attackers guess).
6. Set `onboot: 1` on CT 101 and VM 100, rebooted the host to load kernel `7.0.14-25-pve`, confirmed everything came back on its own.

## Verification

| Check | Command | Result |
|---|---|---|
| Service | `systemctl status jellyfin` | `active (running)`, enabled |
| Listening | `ss -tlnp \| grep 8096` | `0.0.0.0:8096` (jellyfin) |
| GPU groups | `id jellyfin` | `44(video)`, `992(render)` (matches passthrough gid) |
| Boot persistence | `grep onboot` on both configs, then host reboot | CT 101 + VM 100 auto-started |
| Kernel | `uname -r` | `7.0.14-25-pve` |
| Web UI | `http://10.0.0.241:8096` | Logged in as `mike` |
| First media | `scp` from desktop → `fight-film/2026-04-25 Pro Fight.mp4` (60 MB @ 20.8 MB/s) | Visible in Fights library after scan |

## Troubleshooting Log

| Symptom | Diagnosis | Fix |
|---|---|---|
| `apt update` hung on `Ign:` lines, then `Temporary failure resolving` | Bottom-up: route ✅ → gateway ping ✅ → `1.1.1.1` ping ✅ → `resolv.conf` showed `nameserver 100.100.100.100`. **Tailscale MagicDNS rewrote the host's resolv.conf; the container inherited it but isn't on the tailnet** | `pct set 101 --nameserver 75.75.75.75 --searchdomain home.lan`, `pct reboot 101` |
| Commands pasted as a block never ran | Typed while `apt` / `pct enter` was still running, so the new shell never received them | Wait for the prompt; run interactive commands (`pct enter`, `ssh`) on their own |
| `apt full-upgrade` frozen at 87% for 7+ minutes | Second session: `dpkg` at 0% CPU with **no child process** (`ps --forest`), so it wasn't working or waiting on a prompt; it was blocked writing to the screen. **Terminal output paused (XOFF / Ctrl+S)** | `Ctrl+Q` to resume |
| Snapshot of CT 101 failed | Bind mounts can't be snapshotted | Use `vzdump` backups for this container |
| Admin account created as `jellyfin` | Typed in the wrong field | Renamed to `mike` in Dashboard → Users |
| First fight video copied in (`scp`), but library showed "Nothing here" | Ruled out permissions first: in the container the file showed as `nobody:nogroup` (unprivileged mapping) but `-rw-r--r--`, and `runuser -u jellyfin -- ls` listed it, so Jellyfin could read it. Remaining cause: no library scan had run | Dashboard → **Scan All Libraries** |

## Concepts Learned

- **LXC vs VM:** shared kernel vs full virtual hardware. Containers = lighter; VMs = stronger isolation.
- **Bind mounts:** a host folder appears inside the container. Data lives on the host, so rebuilding the container doesn't touch media.
- **Device passthrough + group IDs:** a process can use the GPU only if its group matches the device's group.
- **DNS inheritance:** containers copy the host's DNS by default; a change on the host (Tailscale) silently broke a new container a week later. *When something breaks, ask what changed.*
- **Supply-chain hygiene:** checksum the installer, read what it does, rely on `Signed-By` so apt only accepts packages signed by the vendor's key.
- **UPnP risk:** "automatic port mapping" = an app opening holes in your router. Keep it off.
- **Process states:** `ps --forest` + `%CPU` + `STAT` tell you whether something is working, waiting on input, or blocked.

## Next: Session 3

- Load test media and fix bind-mount file permissions (unprivileged UID mapping)
- Enable Intel QuickSync / VAAPI hardware transcoding; prove it with `intel_gpu_top`
- Install Jellyfin apps on TV and phone; test playback over Tailscale on cellular
