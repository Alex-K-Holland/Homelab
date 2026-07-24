# Storage — TrueNAS SCALE

**Status: Operational. Both pools healthy, encrypted, and in production use.**

## Hardware

- **Chassis:** Jonsbo N4
- **Motherboard:** ASRock N100M (Intel N100)
- **Drives (9 total):**
  - 4× mechanical HDD (bulk pool)
  - 4× SATA SSD (fast pool)
  - 1× 128GB SATA SSD (OS boot drive, external via USB)
- **Storage expansion:** ASM1166-based NVMe-to-6-port SATA expander in the native M.2 Key M slot.

### Hardware quirks (resolved)

1. **M.2 Wi-Fi slot cannot be used for storage.** The board's M.2 Key E 2230 slot is hardwired exclusively for Intel CNVio (integrated WiFi/BT) and has no PCIe storage traces or SATA multiplexers — an M.2-to-dual-SATA adapter card plugged in here was invisible to BIOS and OS. Fix: all storage expansion consolidated onto the ASM1166 NVMe expander in the Key M slot instead.
2. **9 drives, 8 available internal SATA channels.** 2 native SATA ports + 6 from the NVMe expander = 8, one short of the 9 needed. Fix: moved the 128GB boot SSD outside the chassis via a USB 3.0-to-SATA adapter cable.
3. **BIOS wouldn't detect the USB boot drive on rear ports.** The N100M's integrated graphics forces native UEFI and disables CSM when no external GPU is present, so the rear USB controller doesn't initialize early enough in POST. Fix: relocated the boot adapter to the front-panel USB header, which initializes earlier and is reliably detected as a UEFI boot option.

## Pools

### `ssd-pool`

- **Topology:** RAIDZ2, 4 SSDs (~883 GiB usable) — survives 2 simultaneous drive failures.
- **Encryption:** native ZFS encryption, enabled and confirmed unlocked. Recovery key (`dataset_ssd-pool_keys.json`) saved locally.
  - ⚠️ **Open action item:** back this key up somewhere off-NAS (password manager, USB drive, or offsite cloud) — not yet done.
- **Scrub:** completed clean, 0 errors.
- **System Dataset:** located here (confirmed, not on the USB boot drive).
- **Datasets:**
  | Dataset | Purpose |
  |---|---|
  | `apps/` | Application metadata |
  | `apps/nextcloud-config/` | Nextcloud config |
  | `apps/nextcloud-data/` | Primary Nextcloud user files |
  | `apps/nextcloud-db/` | PostgreSQL DB, owned by UID 999 |
  | `apps/jellyfin-config/` | Jellyfin config, owner `apps:apps` mode `770` |
  | `apps/jellyfin-config/jellyfin-cache` | Jellyfin cache, owner `apps:apps` mode `770` |
  | `apps/jellyfin-config/jellyfin-transcode` | Jellyfin transcode scratch, owner `apps:apps` mode `770` |
  | `cloud_data/` | Purpose undocumented — follow-up item |

### `hdd-pool`

- **Topology:** RAIDZ1, 4× 1TB HDDs, ~2.55 TiB usable. RAIDZ1 chosen over RAIDZ2 because important/active work stays on Proxmox's local NVMe — only lower-priority backups live solely here, making RAIDZ1's extra capacity worth the reduced (single-drive) fault tolerance.
- **Encryption:** enabled at creation, recovery key downloaded and saved.
- **Scrub:** scheduled Sundays at 00:00. Auto TRIM: off.
- **Health:** online, no errors, disk temps normal (26–34°C).
- **Datasets:**
  | Dataset | Purpose | Key settings |
  |---|---|---|
  | `media` | Jellyfin library root | Recordsize 1M, atime off, LZ4 compression, owner `apps:apps` |
  | `media/Movies` | Movies (independent ZFS subdataset) | owner `apps:apps`, mode `770` |
  | `media/TV-Shows` | TV shows (independent ZFS subdataset) | owner `apps:apps`, mode `770` |
  | `media/Anime` | Anime (independent ZFS subdataset) | owner `apps:apps`, mode `770` |
  | `files` | General personal file storage | Recordsize 128K (default), atime off, LZ4 |
  | `lab-backups` | Proxmox backup target | Recordsize 1M, atime off, ZSTD compression, quota not yet set |

## Key Learnings

- **ZFS subdatasets don't reliably inherit parent permissions.** `Movies`, `TV-Shows`, and `Anime` defaulted to `750` despite the parent `media` dataset being `770` — verify and explicitly set permissions on every new subdataset.
- **App datasets must be `apps:apps`, not `root:root`.** A dataset silently reverting to root ownership (e.g. after a failed app deploy/cleanup) will break container startup — this caused a Jellyfin segfault crash loop (see `apps.md`).
- **Independently-mounted ZFS subdatasets are invisible to a Docker container that only bind-mounts the parent path.** Each subdataset needs its own explicit mount into the container.
- **TrueNAS app deletion does not delete host path datasets/folders.** Stale config/cache data can persist across reinstalls and needs to be manually verified/cleared.
- This version of TrueNAS SCALE runs apps on **Docker directly**, not k3s/Kubernetes — use `docker ps -a` / `docker logs`, not `k3s kubectl`.

## Deferred / Backlog

- Off-NAS backup of both pools' encryption recovery keys.
- Set a quota on `lab-backups`.
- Configure an actual NFS/SMB share and Proxmox backup job pointing at `lab-backups`.
- Document the purpose of the `cloud_data` dataset on `ssd-pool`.
- Shell alias or fstab/systemd mount unit for the long SMB mount command used for media transfers (see `apps.md`).
