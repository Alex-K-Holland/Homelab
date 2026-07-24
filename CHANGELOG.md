# Changelog

Dated build history of the homelab. Newest entries at the top. This is append-only — past entries are not rewritten as the setup evolves; current-state facts live in `README.md` and `docs/`.

---

## 2026-07-23 — Privacy architecture, WiFi gap identified, Mullvad rollout

- Documented the distinct roles of each privacy layer (local browsing, ISP visibility, IP-based tracking, device-to-device access, fingerprinting, data-at-rest) and clarified that Tailscale and Mullvad are complementary, not redundant.
- Identified that household WiFi is broadcast by the Verizon gateway **upstream** of OPNsense — WiFi clients currently bypass all firewall/VLAN policy and any future router-level VPN.
- Evaluated wireless access points (Cisco Aironet, Ruckus H510, TP-Link Omada EAP670, Ubiquiti U6 Pro/Lite, Ruckus R710, Ruckus R720). **Selected and ordered the Ruckus R710** (901-R710-US00) for its 4x4:4 MU-MIMO / BeamFlex+ and price-to-performance; confirmed a documented path exists to convert it to Unleashed firmware via local web UI if it doesn't arrive pre-loaded.
- Identified and ordered a gigabit 802.3af/at PoE injector (GS308E has no PoE ports).
- Purchased the Tailscale Mullvad VPN add-on (5 device slots). Enabled the Mullvad exit node on the desktop (CLI) and Android phone (app); Windows laptop setup instructions provided, not yet completed.
- Deliberately excluded the NAS from the Mullvad exit node — its traffic is almost entirely inbound and already protected end-to-end by Tailscale's WireGuard encryption.
- Established `curl ipinfo.io` as the standard method to verify exit-node routing.
- Confirmed the JGS524 unmanaged switch is suitable as a future port expander once the GS308E fills up (not yet installed).

## 2026-07-23 — Jellyfin installed and verified end-to-end

- Installed Jellyfin via TrueNAS Apps. Config/cache/transcode placed on `ssd-pool` (low-latency), media on `hdd-pool/media` subdatasets (bulk sequential storage), all under `apps:apps` ownership.
- **Fixed a segfault crash loop:** the `jellyfin-cache` dataset had silently reverted to `root:root` ownership during a prior failed deploy; corrected to `apps:apps`, mode `770`.
- **Fixed an empty-library bug:** Jellyfin's container only bind-mounted the parent `media` path, which doesn't surface independently-mounted ZFS subdatasets (`Movies`, `TV-Shows`, `Anime`). Fixed by adding three explicit per-dataset Additional Storage mounts.
- Created a dedicated `media` SMB-only user (least privilege — SMB access only, no shell/SSH) for pushing ripped media from the desktop, with `apps` as an auxiliary group.
- Fixed subdataset permissions — `Movies`/`TV-Shows`/`Anime` had defaulted to `750` rather than inheriting the parent's `770`, blocking writes.
- Verified the full workflow end-to-end: mounted the SMB share with correct `uid`/`gid`/`file_mode`/`dir_mode` options, copied in a test `.mkv`, scanned the library, confirmed it appeared and played.
- Established the naming convention `Movies/Title (Year)/Title (Year).mkv` for reliable metadata matching.

## 2026-07-12 — Storage pools rebuilt, Nextcloud fully live

- Confirmed network and Proxmox stable with no outstanding work.
- Rebuilt `ssd-pool` clean: RAIDZ2 across 4 SSDs, ZFS encryption enabled and confirmed unlocked, recovery key saved, scrub completed with zero errors. Confirmed the TrueNAS System Dataset already lived here.
- Built `hdd-pool`: RAIDZ1 across 4× 1TB HDDs (~2.55 TiB usable), encrypted, scrub scheduled for Sundays. Created and permissioned the `media`, `files`, and `lab-backups` datasets.
- Installed and fully configured **Nextcloud** on `ssd-pool`: fixed trusted-domains via `occ` shell command (required since the TrueNAS chart doesn't expose this in the GUI), enabled phone auto-upload, finalized the folder structure, and completed a ~20GB Google Photos migration (Takeout export + Czkawka hash-based dedup, album folders protected as reference).
- Confirmed **Tailscale** live and providing remote access to Nextcloud from phone and desktop.
- Decided desktop access to Nextcloud would be via the web UI directly (not the desktop sync client, to avoid local mirroring).
- Jellyfin planned as the next task; not yet installed.

## Early July 2026 — Proxmox networked, NAS hardware fully diagnosed

- Brought Proxmox VE online: static IP on `vmbr0`, DNS configured (OPNsense primary, Cloudflare secondary), fixed `401 Unauthorized` apt errors by switching to the PVE/Ceph No-Subscription repositories, fully updated.
- Fully diagnosed the TrueNAS NAS hardware (Jonsbo N4 chassis, ASRock N100M, 9 drives total):
  - Found the motherboard's M.2 Wi-Fi slot is CNVio-only and has no PCIe storage traces — consolidated all storage expansion onto an ASM1166-based NVMe-to-6-port SATA expander instead.
  - Resolved a 9-drive/8-port bottleneck by moving the 128GB boot SSD external via USB.
  - Resolved a BIOS POST failure to detect the USB boot drive by relocating it to the front-panel USB header (rear ports don't initialize early enough without CSM).
  - Confirmed all 8 internal pool drives (4 HDD, 4 SSD) visible and healthy.
- Found `ssd-pool` in a degraded/unhealthy state due to stale metadata from when two drives were locked out by the Wi-Fi slot issue — flagged for a full export/destroy and rebuild (completed in the 2026-07-12 session above).
- `hdd-pool` not yet built at this point.

## 2026-06-30 — Initial buildout

- OPNsense installed and running headlessly as router/firewall (router-on-a-stick), behind an upstream Verizon gateway (double-NAT).
- Netgear managed switch configured with VLAN tagging to separate WAN and LAN traffic.
- NAS chassis assembled with all drives installed; SSD pool formatted and aggregated; HDD drives recognized but not yet pooled.
- Proxmox VE installed as an offline base install — not yet connected to the network.
- Roadmap laid out: connect Proxmox, provision HDD pool, deploy Nextcloud/media server, set up Tailscale.
