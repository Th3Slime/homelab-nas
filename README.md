# 🖥️ Homelab NAS — Repurposed Desktop → Self-Hosted Server

Turning a spare desktop into a dedicated network-attached storage (NAS) and self-hosting platform running Unraid — built as a hands-on lab for storage, containers, networking, and cybersecurity.

## 📌 Overview

I took an ageing desktop PC that was sitting unused and rebuilt it into a dedicated home server. Rather than buying an off-the-shelf NAS, I went the DIY route to learn the underlying infrastructure — storage architecture, Linux administration, containerisation, networking, and security — from the ground up.

The result is a low-cost, self-hosted server that replaces several paid cloud subscriptions and doubles as a practical lab environment for IT and cybersecurity skill-building.

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
| Drives | 4 × 1 TB 2.5" SATA HDD (expandable — 6 onboard + 6 card ports) |
| Boot | USB flash drive (Unraid boots from USB by design) |
| Cache | 2.5" SATA SSD for app/container data (planned) |
| PSU | 525 W |

**Design note:** the motherboard's 6 SATA ports were a hard limit, so I added a PCIe HBA to scale drive count — a deliberate choice of a plain AHCI controller over a hardware-RAID card, since Unraid manages redundancy itself.

## 🗄️ Storage architecture

- **Array:** individual data disks with a dedicated parity disk for fault tolerance — a single drive failure can be rebuilt without data loss.
- **Cache pool (planned):** SSD tier for Docker app data and active writes, keeping the spinning disks quiet and apps responsive.
- **Mixed-drive friendly:** Unraid's approach allows drives of different sizes to be pooled and expanded over time — ideal for a lab that grows incrementally.

```
        ┌──────────────────────────────┐
        │           UNRAID OS          │  ← boots from USB
        ├──────────────┬───────────────┤
        │   ARRAY      │   CACHE (SSD)  │
        │  parity +    │  Docker /      │
        │  data disks  │  app data      │
        └──────┬───────┴───────┬────────┘
        onboard SATA ×6     ASM1166 PCIe HBA ×6
```

## 📦 Services (self-hosted stack)

Deployed / planned as Docker containers via Unraid's Community Applications:

| Service | Purpose | Replaces |
|---|---|---|
| Pi-hole | Network-wide ad & tracker blocking (DNS) | — |
| Immich | Photo & video backup with AI search | Google Photos / iCloud |
| Jellyfin | Media streaming (hardware transcoding via QuickSync) | Plex / streaming subs |
| Vaultwarden | Self-hosted password manager | 1Password / paid Bitwarden |
| Nextcloud | File sync, calendar, contacts | Dropbox / Google Drive |
| Tailscale | Secure remote access (zero-trust VPN) | Exposed ports |

## 🧠 Skills demonstrated

- **Linux server administration** — Unraid (Slackware-based), CLI, services, boot process
- **Storage & data protection** — parity/redundancy concepts, array management, SMART monitoring
- **Hardware & troubleshooting** — PCIe/SATA expansion, controller selection, drive diagnostics, power/thermal planning
- **Containerisation** — Docker-based service deployment and management
- **Networking** — DHCP, DNS, static reservations, reverse proxy (planned), VLAN segmentation (planned)
- **Security mindset** — least-exposure design, VPN-based remote access instead of port forwarding, credential hygiene, notifications & monitoring
- **Documentation** — repeatable build notes and a maintained roadmap

## 🗺️ Roadmap

**Done**

- [x] Unraid install & first boot (USB)
- [x] PCIe SATA HBA installed and detected
- [x] Initial array with parity protection
- [x] Healthy SMART status across all drives

**In progress / planned**

- [ ] RAM upgrade 8 GB → 32 GB
- [ ] SSD cache pool for containers
- [ ] Expand array with additional drives
- [ ] Deploy core services (Pi-hole, Immich, Jellyfin, Vaultwarden)
- [ ] Tailscale for secure remote access
- [ ] Grafana + Prometheus monitoring dashboards
- [ ] Nginx Proxy Manager (reverse proxy + TLS)
- [ ] VLAN network segmentation
- [ ] Security lab: isolated VMs (Kali + intentionally vulnerable targets) for hands-on pentesting practice

## 📝 Lessons learned

- **Match the controller to the job** — a plain AHCI HBA beats a hardware-RAID card when the OS handles redundancy.
- **Plan around the platform's limits** — SATA port count, RAM ceiling, and licensing device caps all shaped the design.
- **Security first, exposure last** — remote access belongs behind a VPN, never a forwarded port.
- **Cheap ≠ limited for learning** — old hardware is a fantastic, low-risk platform to break things and understand why they break.

## 🔒 Note

This repository documents a personal home lab. All credentials, keys, internal IP addresses, and other sensitive details have been intentionally omitted. Nothing here should be treated as production configuration.

---

Built and documented as an ongoing learning project. Feedback and homelab war stories welcome — open an issue or reach out.
