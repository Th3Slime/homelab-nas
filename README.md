# 🖥️ Homelab NAS — Repurposed Desktop → Self-Hosted Server

Turning a spare desktop into a dedicated network-attached storage (NAS) and self-hosting platform running Unraid — built as a hands-on lab for storage, containers, networking, and cybersecurity.

## 📌 Overview

I took an ageing desktop PC that was sitting unused and rebuilt it into a dedicated home server. Rather than buying an off-the-shelf NAS, I went the DIY route to learn the underlying infrastructure — storage architecture, Linux administration, containerisation, networking, and security — from the ground up.

The result is a low-cost, self-hosted server that replaces several paid cloud subscriptions and doubles as a practical lab environment for IT and cybersecurity skill-building.

**Status:** live and in daily use. ✅ Array healthy: 7 drives active (1 parity + 6 data), all passing SMART. ✅ All eight services are running in production. ⚪ The Disk 2 slot is empty after a drive failure and is waiting for a replacement. The paid cloud photo subscription the server was built to replace has been cancelled. The latest recovery is written up in [Incidents](#-incidents).

## 🎯 Why I built it

- **Own my data** — self-hosted photo backup and file sync instead of monthly cloud fees.
- **Learn by doing** — a real, always-on system is the best way to build practical infrastructure and security skills that don't come from tutorials alone.
- **Resourcefulness** — reuse old hardware and salvaged drives instead of buying new.

## 🧰 Hardware

| Component | Detail |
|---|---|
| Case | BitFenix mid-tower (repurposed) |
| Motherboard | Gigabyte GA-B85-HD3-A (LGA1150, Intel B85) |
| CPU | Intel Core i5-4460 (4C/4T, Haswell) — HD 4600 iGPU w/ QuickSync |
| RAM | 8 GB DDR3 (upgrade to 32 GB planned) |
| Storage controller | ASM1166 6-port PCIe SATA card (non-RAID / AHCI HBA) |
| Drives | 7 × 1 TB 2.5" SATA HDD active — 1 parity + 6 data. Disk 2 slot empty after a drive failure, replacement pending (expandable — 6 onboard + 6 card ports) |
| Boot | USB flash drive (Unraid boots from USB by design) |
| Cache | 2.5" SATA SSD for app/container data (planned) |
| PSU | 525 W |
| Licence | Unraid OS Unleashed — unlimited attached devices |

**Design note:** the motherboard's 6 SATA ports were a hard limit, so I added a PCIe HBA to scale drive count — a deliberate choice of a plain AHCI controller over a hardware-RAID card, since Unraid manages redundancy itself. The licence tier was the second ceiling: the entry tier caps attached devices, so expanding past six meant moving up to Unleashed. Two limits, two different kinds of fix.

## 🗄️ Storage architecture

- **Array:** 7 active drives — 1 dedicated parity disk + 6 data disks, ~6 TB usable. The Disk 2 slot is empty after a controller/firmware failure; parity was resynced without it, so the array is fully protected while the replacement is pending. A single drive failure can be rebuilt without data loss.
- **Expanded live:** the array grew from 4 drives to 8 while in service. Each expansion meant stopping the array, assigning the new disk, and letting parity rebuild — routine on paper, and a genuinely useful thing to have done under real conditions.
- **Cache pool (planned):** SSD tier for Docker app data and active writes, keeping the spinning disks quiet and apps responsive. Currently the highest-priority upgrade — container data living on the array is the main performance bottleneck.
- **Filesystem migration (in progress, high priority):** two disks carried over as NTFS from a previous life and are being migrated to XFS to bring the array to a consistent filesystem. Docker system storage and appdata currently sit on one of them. An unclean shutdown on that NTFS disk caused the [September 2026 Immich outage](docs/incidents/2026-09-25-disk-failure-recovery.md), so moving them off NTFS comes first.
- **Dual parity (planned):** a second parity disk, now that the licence allows unlimited devices.

```
┌──────────────────────────────┐
│ UNRAID OS │ ← boots from USB
├──────────────┬───────────────┤
│ ARRAY │ CACHE (SSD) │
│ 1 parity + │ Docker / │
│ 6 data │ app data │
│ ~6 TB usable │ (planned) │
└──────┬───────┴───────┬────────┘
onboard SATA ×6 ASM1166 PCIe HBA ×6
```

## 📦 Services (self-hosted stack)

Deployed as Docker containers via Unraid's Community Applications:

| Service | Purpose | Replaces | Status |
|---|---|---|---|
| Pi-hole | Network-wide ad & tracker blocking (DNS) | — | ✅ Live |
| Immich | Photo & video backup with AI search | Google Photos / iCloud | ✅ Live |
| Jellyfin | Media streaming (hardware transcoding via QuickSync) | Plex / streaming subs | ✅ Live |
| Tailscale | Secure remote access (zero-trust VPN) | Exposed ports | ✅ Live |
| Nginx Proxy Manager | Reverse proxy + TLS for internal services | — | ✅ Live |
| Grafana | Metrics dashboards for system and services | — | ✅ Live |
| Prometheus | Metrics collection feeding Grafana | — | ✅ Live |
| Uptime Kuma | Uptime monitoring and alerting | — | ✅ Live |
| Vaultwarden | Self-hosted password manager | 1Password / paid Bitwarden | ⏳ Planned |
| Nextcloud | File sync, calendar, contacts | Dropbox / Google Drive | ⏳ Planned |
| Home Assistant | Home automation | — | ⏳ Planned |

## 💰 Cost impact

The paid cloud photo storage subscription has been cancelled — Immich now handles automatic phone photo backup, album management, and search on hardware I already own. That was the first measurable return on the build, and the one that justified the rest of it.

## 🧠 Skills demonstrated

- **Linux server administration** — Unraid (Slackware-based), CLI, services, boot process
- **Storage & data protection** — parity/redundancy concepts, live array expansion, degraded-array recovery, SMART monitoring
- **Hardware & troubleshooting** — PCIe/SATA expansion, controller selection, drive diagnostics, power/thermal planning
- **Containerisation** — Docker deployment and management across eight production services (11 containers), volume and persistent-data management, log-driven debugging
- **Container networking** — bridge vs macvlan networking modes and when each is appropriate; diagnosing port conflicts and IP allocation
- **Networking** — DHCP, DNS, static reservations, reverse proxy + TLS termination, mesh VPN, VLAN segmentation (planned)
- **Monitoring & observability** — metrics dashboards and uptime alerting across the stack
- **Security mindset** — least-exposure design, VPN-based remote access instead of port forwarding, credential hygiene, notifications & monitoring
- **Documentation** — repeatable build notes and a maintained roadmap

## 🔧 Troubleshooting log

Real problems hit and solved. This is the part of the project I've learned the most from.

**Pi-hole wouldn't resolve DNS on the default bridge network.** Port 53 was already bound on the host. The fix was moving Pi-hole to a macvlan network, giving the container its own MAC address and IP directly on the LAN rather than sharing the host's network stack. Understanding why bridge and macvlan behave differently — and that a DNS server binding a privileged port is exactly the case where macvlan earns its keep — was the single most useful thing I learned in this project.

**Immich wouldn't accept logins on first deployment.** Worked the problem through container logs and service dependencies rather than tearing the stack down and reinstalling. Resolved once dependent services had fully initialised — a lesson in container start-up ordering, and in reading logs before reaching for a rebuild.

**The GUI wasn't enough.** Several faults were only diagnosable from the command line, where the web interface gave incomplete or misleading output. Dropping to the shell to inspect container state, network interfaces, and logs directly became routine rather than a last resort.

**A data disk dropped out of the array.** Because the array is parity-protected, the disk's contents remained readable as an emulated device while the physical drive was diagnosed. Working through it reinforced the practical difference between redundancy and backup — parity buys you time and a rebuild path, it doesn't replace a copy of your data elsewhere.

**Immich's database wouldn't start after an unclean shutdown.** Postgres failed with `Permission denied` on its config file. The error path pointed inside the container rather than at the data directory, so I backed up the database first and then traced the fault to the container's writable layer on an NTFS disk. Recreating the container from its template fixed it, and neither the image nor the database needed repairing. [Full write-up →](docs/incidents/2026-09-25-disk-failure-recovery.md)

## 🚨 Incidents

Formal write-ups of outages and recoveries, covering timeline, diagnosis, root cause, fix and follow-up actions.

| Date | Incident | Impact | Status |
|---|---|---|---|
| 2026-09-25 | [Disk 2 failure and Immich database recovery](docs/incidents/2026-09-25-disk-failure-recovery.md) | Immich unavailable. No data lost | ✅ Resolved. Disk 2 replacement pending |

## 🗺️ Roadmap

**Done**

- [x] Unraid install & first boot (USB)
- [x] PCIe SATA HBA installed and detected
- [x] Initial array with parity protection
- [x] Healthy SMART status across all drives
- [x] Array expanded from 4 to 8 drives with parity rebuild
- [x] Upgraded to Unraid Unleashed licence (unlimited devices)
- [x] Pi-hole deployed — network-wide DNS ad blocking
- [x] Immich deployed — cloud photo subscription cancelled
- [x] Jellyfin deployed with QuickSync hardware transcoding
- [x] Tailscale deployed for secure remote access
- [x] Nginx Proxy Manager deployed — reverse proxy + TLS
- [x] Grafana dashboards live
- [x] Uptime Kuma monitoring live
- [x] Prometheus alongside Grafana for metrics collection
- [x] Diagnose and remove failed Disk 2
- [x] Recover Immich database after unclean shutdown

**In progress / planned**

- [ ] Migrate appdata and system shares off NTFS Disk 1 onto XFS, then convert Disks 1 and 3 to XFS — high priority
- [ ] Scheduled Immich database backup script — high priority
- [ ] SSD cache pool for containers — top priority
- [ ] Unplug the failed drive and install the replacement 1 TB Disk 2
- [ ] Add a 4 TB drive — Unraid parity must be at least as large as the biggest data disk, so the 4 TB drive goes in as the new parity disk first, with the old 1 TB parity reused as a data disk
- [ ] SMART monitoring script, including load-cycle count tracking
- [ ] RAM upgrade 8 GB → 32 GB
- [ ] Second parity disk for dual-parity redundancy
- [ ] Roll out Pi-hole as DNS across every device on the network
- [ ] Remote Immich access via Tailscale
- [ ] Deploy Vaultwarden and Nextcloud
- [ ] Home Assistant for home automation
- [ ] UPS for clean shutdown on power loss
- [ ] VLAN network segmentation
- [ ] Security lab: isolated VMs (Kali + intentionally vulnerable targets) for hands-on pentesting practice

## 📝 Lessons learned

- **Match the controller to the job** — a plain AHCI HBA beats a hardware-RAID card when the OS handles redundancy.
- **Plan around the platform's limits** — SATA port count, RAM ceiling, and licensing device caps all shaped the design.
- **Drive choice matters more than drive count** — the 2.5" drives in this build are SMR, which rewrite entire shingled zones on every write. They're cheap and plentiful, but they make parity rebuilds slow and raise the odds of a drive timing out mid-rebuild. Future expansion will use CMR.
- **Redundancy is not backup** — parity survives a disk failure. It doesn't survive a mistake, a fire, or a delete. Knowing the difference is the point.
- **Security first, exposure last** — remote access belongs behind a VPN, never a forwarded port.
- **The GUI is a convenience, not the source of truth** — every hard problem in this build was solved on the command line.
- **Keep container storage on a Linux-native filesystem** — NTFS under Linux handles unclean shutdowns badly, and one crash became a database outage.
- **Read the error, back up, then act** — the error path pointed straight at the real fault, and a backup before any change makes every next step reversible.
- **Cheap ≠ limited for learning** — old hardware is a fantastic, low-risk platform to break things and understand why they break.

## 🔒 Note

This repository documents a personal home lab. Hostnames, internal IP addresses and subnets, drive serial numbers, custom or identifying names (such as user-created shares and Docker networks), licence identifiers, credentials, keys, and configuration files have all been intentionally omitted. Standard Unraid and Immich default paths (such as `appdata` and `system`) are kept, since they're the same on every install and help explain the troubleshooting. Nothing here should be treated as production configuration.

The reasoning is the transferable part — anyone reproducing this build needs the decisions, not my specific values.

---

Built and documented as an ongoing learning project. Feedback and homelab war stories welcome — open an issue or reach out.
