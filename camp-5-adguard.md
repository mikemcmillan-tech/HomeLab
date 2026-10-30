# Camp 5: AdGuard Home (Network DNS Filtering)

- **Dates:** 2026-10-29 →
- **Status:** ⏳ In progress. Sessions 1–2 done; Session 3 (device rollout) next
- **Goal:** Block ads and trackers at the DNS level, without risking the household's internet.
- **Finish line:** Blocked domains return `0.0.0.0`, and my phone gets filtering on cellular through Tailscale.

---

## Rollout Strategy: Rings, Not a Big Bang

The usual setup points the router's DHCP at AdGuard so every device in the house uses it. I don't have router admin access, and if AdGuard went down, the whole family would lose internet. So the rollout happens in rings:

| Ring | Who uses AdGuard | Blast radius if it breaks |
|---|---|---|
| 1 | My laptop (manual DNS) | Just me; 30-second rollback |
| 2 | My phone, anywhere, via Tailscale DNS | Just my tailnet devices |
| 3 | Whole house via router | Family. Only with router access + a written rollback plan |

## Architecture

| Item | Value |
|---|---|
| Container | CT 102 `adguard`, unprivileged Debian 13 LXC |
| Resources | 1 core, 512 MB RAM, 4 GB disk (DNS is lightweight; right-sized, not over-allocated) |
| IP | `10.0.0.242/24` (static), gateway `10.0.0.1` |
| Services | DNS on `10.0.0.242:53` (TCP/UDP), admin UI on `10.0.0.242:80` |
| Upstreams | DNS over HTTPS: Quad9 + Cloudflare (bootstrap `9.9.9.9`, `1.1.1.1`) |
| Exposure | LAN + Tailscale only. **Port 53 is never forwarded** (open resolvers get abused for DNS amplification DDoS) |

## Session 1: Fix Host DNS + Build CT 102

1. **Fixed a leftover from Camp 4.** Tailscale had rewritten the Proxmox host's `/etc/resolv.conf` to `100.100.100.100` (MagicDNS), and every new container inherited it. That's what broke DNS in the Jellyfin container. The host is a subnet router and doesn't need Tailscale's DNS:
   ```bash
   tailscale set --accept-dns=false
   ```
   Result: host back on `75.75.75.75`, subnet route still working. Rollback: `tailscale set --accept-dns=true`.
2. **Built the container from the CLI** instead of the GUI wizard, so the build is one repeatable command:
   ```bash
   pveam list hdd-storage
   pct create 102 hdd-storage:vztmpl/debian-13-standard_13.6-1_amd64.tar.zst \
     --hostname adguard --unprivileged 1 --features nesting=1 \
     --cores 1 --memory 512 --swap 512 --rootfs local-lvm:4 \
     --net0 name=eth0,bridge=vmbr0,ip=10.0.0.242/24,gw=10.0.0.1 \
     --onboot 1
   ```
3. **Proof the host fix worked:** CT 102 inherited `nameserver 75.75.75.75` with no manual fix (CT 101 had needed one).
4. Updated, then snapshot `clean-install`.

## Session 2: Install + Lock Down

1. Downloaded AdGuard's official install script and reviewed it with `grep` before running it (downloads only from AdGuard's servers, installs to `/opt/AdGuardHome`, creates a systemd service).
2. Setup wizard: admin UI on port 80, DNS on port 53, non-default admin user.
3. Switched upstreams to **DNS over HTTPS** so queries leaving my network are encrypted.
4. **Bound services to the LAN address only.** The wizard's "All interfaces" also listened on the container's **public IPv6 address** (`2601:...`). IPv6 has no NAT, and I can't verify the ISP router's IPv6 firewall, so I didn't rely on it. Edited the config directly:
   ```bash
   cp /opt/AdGuardHome/AdGuardHome.yaml /root/AdGuardHome.yaml.bak
   systemctl stop AdGuardHome
   sed -i 's|address: 0.0.0.0:80|address: 10.0.0.242:80|' /opt/AdGuardHome/AdGuardHome.yaml
   sed -i '/bind_hosts:/{n;s/0\.0\.0\.0/10.0.0.242/}' /opt/AdGuardHome/AdGuardHome.yaml
   systemctl start AdGuardHome
   ```
   This closes both the DNS port **and** the admin login page to the internet (an "allowed clients" list would only have protected DNS).
5. Set the container timezone so query logs show local time.
6. Snapshot `adguard-configured`, then rebooted the host onto kernel `7.0.14-26-pve`.

## Verification

| Check | Command | Result |
|---|---|---|
| Blocking | `nslookup doubleclick.net 10.0.0.242` | `0.0.0.0` and `::` (blocked on IPv4 and IPv6) |
| Normal resolution | `nslookup google.com 10.0.0.242` | Real Google IPs |
| Binding | `ss -tulnp \| grep -E ':53 \|:80 '` | `10.0.0.242:53` (tcp+udp), `10.0.0.242:80`; no `*` |
| Service | `systemctl status AdGuardHome` | `active (running)`, enabled |
| Survives reboot | Host reboot → `pct list; qm list` | CT 101, CT 102, VM 100 all auto-started |

Testing with `nslookup <name> 10.0.0.242` queries AdGuard directly without changing any device's DNS, so the test has zero blast radius.

## Troubleshooting Log

| Symptom | Diagnosis | Fix |
|---|---|---|
| `pct create` failed: `can't find file 'TEMPLATE'` | The placeholder was pasted instead of the real template name. Proxmox created the disk, failed, then removed it, so nothing was left half-built | `pveam list hdd-storage` to get the exact name, then reran |
| Thin pool `WARNING`s during create | Thin provisioning: total disk promised to guests (~152 GiB) exceeds the physical pool. Fine while disks are mostly empty | Monitor pool usage (Camp 7) |
| `ss` showed `*:53` / `*:80` after the wizard | "All interfaces" included a public IPv6 address | Bound to `10.0.0.242` in `AdGuardHome.yaml` |
| `ss` printed nothing right after `systemctl start` | Ran before the service finished opening ports | `systemctl status` showed it running; reran `ss` |
| Logs in UTC | Container timezone not set | `timedatectl set-timezone America/Los_Angeles` |

## Concepts Learned

- **DNS sinkholing:** answer `0.0.0.0` for ad and tracker domains so the request never leaves the device.
- **Port 53** (DNS, TCP + UDP), **port 80** (HTTP).
- **DNS over HTTPS (DoH):** encrypted DNS. Bootstrap servers are the plain-DNS addresses used once to find the DoH servers.
- **IPv6 has no NAT:** devices can have globally routable addresses, so binding and firewalls matter more.
- **Bind to the interface you mean:** listening on `0.0.0.0` / `*` means every address, including ones you didn't think about.
- **Change rings / blast radius:** roll out to yourself first, then expand.
- **Infrastructure as commands:** a CLI `pct create` is repeatable and documentable in a way GUI clicks aren't.

## Next: Session 3

- Ring 1: point my laptop's DNS at `10.0.0.242`, with a rollback.
- Ring 2: set AdGuard as the DNS server for my tailnet so my phone filters on cellular.
- Review the query log and allowlist anything that breaks.
