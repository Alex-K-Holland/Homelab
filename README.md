# Homelab

A privacy-focused, self-hosted homelab: local media and file services, ISP-level privacy, and secure remote access to every personal device — no public port forwarding anywhere.

This README is a living snapshot of the **current state**. For the session-by-session history (including bugs hit and how they were fixed), see [`CHANGELOG.md`](./CHANGELOG.md). For topic-specific detail, see [`docs/`](./docs).

## Goals

- Self-hosted media and file services (own the data, no reliance on paid cloud tiers)
- Comprehensive internet privacy, from the ISP layer down
- Secure remote access across all personal devices without exposing anything to the public internet
- No paywalled features; media library built exclusively from owned/ripped physical media

## Stack Overview

| Layer | Tool | Status |
|---|---|---|
| Router / Firewall | OPNsense | ✅ Operational |
| Switching | Netgear GS308E (managed, VLANs) | ✅ Operational |
| Switching (expansion) | Netgear JGS524 (unmanaged) | ⏳ Not yet installed |
| Wireless AP | Ruckus R710 (Unleashed) | ⏳ Ordered, not yet installed |
| Hypervisor | Proxmox VE | ✅ Operational |
| NAS / Storage | TrueNAS SCALE | ✅ Operational |
| File sync / photos | Nextcloud | ✅ Live |
| Media server | Jellyfin | ✅ Live |
| Remote access mesh | Tailscale (WireGuard) | ✅ Live |
| ISP/IP privacy | Mullvad (via Tailscale exit node) | ✅ Live (desktop, phone); laptop pending |
| CCNA lab environment | Proxmox (planned) | ⏳ Not started |

## Architecture at a Glance

```
Internet
   │
Verizon Gateway (double-NAT, currently also broadcasts WiFi — see docs/network.md)
   │
OPNsense (router/firewall, DHCP)
   │
Netgear GS308E (managed switch, VLAN 10 WAN / VLAN 20 LAN)
   ├── Proxmox VE (hypervisor, static IP)
   │      └── CCNA lab environment (planned next)
   ├── TrueNAS SCALE (NAS)
   │      ├── ssd-pool  → apps, Nextcloud, Jellyfin config/cache
   │      └── hdd-pool   → bulk media, personal files, lab backups
   └── Desktop / other LAN clients

Tailscale mesh (WireGuard) ties together: desktop, phone, laptop, NAS
Mullvad exit node (via Tailscale) on: desktop, phone (laptop pending)
```

## Current State Summary

**Network** — OPNsense fully operational behind a Verizon double-NAT. Managed switch with VLAN separation. WiFi currently bypasses OPNsense (broadcast from the Verizon gateway) — a dedicated access point (Ruckus R710) has been selected and ordered to close this gap. See [`docs/network.md`](./docs/network.md).

**Compute** — Proxmox VE installed, networked, and fully updated via community no-subscription repos. Currently hosts nothing beyond base install; next major project is standing up a CCNA lab environment here.

**Storage** — TrueNAS SCALE on a Jonsbo N4 / ASRock N100M build. Two encrypted ZFS pools: `ssd-pool` (RAIDZ2, app data) and `hdd-pool` (RAIDZ1, bulk media/files). See [`docs/storage.md`](./docs/storage.md).

**Apps** — Nextcloud (personal cloud / Google Photos replacement) and Jellyfin (media server, owned/ripped content only) are both live and accessed remotely over Tailscale. See [`docs/apps.md`](./docs/apps.md).

**Privacy & Remote Access** — Tailscale provides the device mesh; Mullvad (via Tailscale's exit-node add-on) hides traffic from the ISP and masks IP from external sites. See [`docs/privacy-vpn.md`](./docs/privacy-vpn.md).

## On the Horizon

- **Next major project:** stand up a CCNA lab environment on the Proxmox node (VMs/containers for routing, switching, and network simulation practice). Details will land in `docs/lab-ccna.md` as the work begins.
- Receive and install the Ruckus R710 access point + PoE injector; convert to Unleashed firmware if needed; migrate WiFi off the Verizon gateway.
- Bring the JGS524 unmanaged switch online once the GS308E fills up.
- OPNsense-level Mullvad WireGuard tunnel + kill switch for whole-home coverage (e.g. smart TV).
- Encrypted DNS (DNS-over-TLS via Unbound) at the router.
- Run WebRTC/DNS leak tests to confirm no leaks.

Full outstanding item list lives in each `docs/` file and in `CHANGELOG.md`.

## Repo Structure

```
homelab/
├── README.md              ← you are here
├── CHANGELOG.md           ← dated build history
├── docs/
│   ├── network.md
│   ├── storage.md
│   ├── apps.md
│   ├── privacy-vpn.md
│   └── lab-ccna.md        ← placeholder, work not yet started
└── hardware/
    └── parts-list.md
```
