# Applications

**Status: Nextcloud and Jellyfin both live. Tailscale provides remote access to both.**

## Nextcloud — Personal Cloud / Google Photos Replacement

**Status: Live, fully operational.**

- Backed by `ssd-pool`, encrypted at rest.
- Accessed via web UI at `http://<tailscale-ip>:30027` (desktop) and the official mobile app (phone), both over Tailscale.
- **Trusted domains** had to be set via the `occ` shell command (the TrueNAS chart doesn't expose this field in the GUI) — required or the browser throws SSL/"malformed server configuration" errors:
  ```
  php occ config:system:set trusted_domains <n> --value="<tailscale-ip>"
  ```
- **Auto-upload** enabled on phone — new camera roll photos/videos sync automatically into the `Camera` folder.

### Folder structure (finalized)

```
Nextcloud root/
├── Camera        (auto-upload target)
├── Great falls et al
├── Japan 2025-26
├── Mexico 2023-24
├── Mexico 24-25
├── Progress
├── smth
├── Turkey
├── UMD
```

All album folders and `Camera` sit as siblings at the root.

### Google Photos migration (complete)

- ~20GB exported via Google Takeout (Photos only).
- Per-photo `.json` metadata sidecar files were verified unnecessary (EXIF "date taken" was already correct) and bulk-deleted rather than merged.
- Deduplicated with **Czkawka** (Blake3 hash-based dedup). Named album folders were set as reference (protected from deletion); the generic "Photos from 20XX" year-dump folders were not, so their duplicates (already present in named albums) were safely bulk-deleted.
  - Note: Czkawka always scans every added folder regardless of reference status — checking "reference" only protects a folder from being selected for deletion.
  - Note: Czkawka's custom-select path matching doesn't support shell-style wildcards; use the built-in reference-folder exclusion instead of trying to glob paths.

### Desktop access — decision

The desktop **sync client** was evaluated and rejected (it mirrors files locally, which conflicts with the "stored only on the NAS" goal). A WebDAV mount was identified as a viable alternative but not set up. Current approach: browse directly via the **web UI**, which meets "accessible everywhere, stored only on the NAS" with zero local footprint.

### Known limitation

`.mov`/`.mp4` playback frequently fails (black screen, no error) in the web UI on desktop Brave — believed to be a browser HEVC codec limitation, not server-side. Not a priority; Nextcloud is used almost exclusively via mobile, where playback works fine.

---

## Jellyfin — Media Server

**Status: Live, verified end-to-end with a real media file.**

### Storage layout

| Purpose | Location | Notes |
|---|---|---|
| Config | `ssd-pool/apps/jellyfin-config` | Owner `apps:apps`, mode `770` |
| Cache | `ssd-pool/apps/jellyfin-config/jellyfin-cache` | Separate dataset, `apps:apps`, `770` |
| Transcode | `ssd-pool/apps/jellyfin-config/jellyfin-transcode` | Separate dataset, `apps:apps`, `770` |
| Media (Movies) | `hdd-pool/media/Movies` | Independent ZFS subdataset |
| Media (TV Shows) | `hdd-pool/media/TV-Shows` | Independent ZFS subdataset |
| Media (Anime) | `hdd-pool/media/Anime` | Independent ZFS subdataset |

Config/cache/transcode sit on `ssd-pool` for low-latency small-file/scratch I/O; media sits on `hdd-pool` for bulk sequential storage.

### Issues encountered & fixed

**1. Segfault crash loop (exit code 139) on first install**
The `jellyfin-cache` dataset had silently been recreated as `root:root` / `755` during a prior failed deploy — Jellyfin (UID 568) couldn't write to its own cache dir. Fixed with:
```bash
sudo chown apps:apps /mnt/ssd-pool/apps/jellyfin-config/jellyfin-cache
sudo chmod 770 /mnt/ssd-pool/apps/jellyfin-config/jellyfin-cache
```
Diagnosed using `docker ps -a` / `docker logs` (this TrueNAS SCALE version runs apps on Docker directly, not k3s). Going forward, use the app wizard's **"Create Dataset"** option for cache/transcode paths rather than typing a raw host path — it sets `apps:apps` ownership automatically.

**2. Empty library despite files present on disk**
`Movies`/`TV-Shows`/`Anime` are independently mounted ZFS subdatasets under `hdd-pool/media`. A single bind mount at the parent path doesn't pull in filesystems mounted underneath it, so the subdatasets appeared empty inside the container. Fixed by replacing the parent-level mount with three explicit Additional Storage mounts (`/mnt/hdd-pool/media/Movies` → `/media/Movies`, etc.).

### SMB access for file transfer

A dedicated `media` user was created for pushing content from the desktop:
- **SMB Access only** — TrueNAS/Shell/SSH access all disabled (least privilege).
- Primary group `media`; auxiliary group `apps` (grants access to `apps:apps`-owned datasets without extra ACLs).
- SMB share: `media` → `/mnt/hdd-pool/media`.

**Verified working mount command:**
```bash
sudo mkdir -p /mnt/nas-media
sudo mount -t cifs //<tailscale-ip>/media /mnt/nas-media \
  -o username=media,vers=3.0,uid=$(id -u),gid=$(id -g),file_mode=0770,dir_mode=0770
```
`mount.cifs` maps all file ops to UID/GID 0 (root) by default regardless of the authenticated SMB user — the explicit `uid`/`gid`/`file_mode`/`dir_mode` options are required for the local desktop user to get correct write permissions.

The `Movies`/`TV-Shows`/`Anime` subdatasets also needed their group permissions explicitly set to `770` (they defaulted to `750` and didn't inherit the parent's `770`).

### Full ingest workflow (verified)

```bash
# 1. Mount the share (if not already mounted)
sudo mkdir -p /mnt/nas-media
sudo mount -t cifs //<tailscale-ip>/media /mnt/nas-media \
  -o username=media,vers=3.0,uid=$(id -u),gid=$(id -g),file_mode=0770,dir_mode=0770

# 2. Create the movie folder (naming convention: "Title (Year)")
mkdir -p '/mnt/nas-media/Movies/Movie Title (Year)'

# 3. Copy the file in, matching folder and file name
cp ~/Downloads/yourfile.mkv '/mnt/nas-media/Movies/Movie Title (Year)/Movie Title (Year).mkv'

# 4. Verify
ls -la '/mnt/nas-media/Movies/Movie Title (Year)/'

# 5. Trigger a scan in the Jellyfin web UI (Dashboard → Libraries → Scan Library)

# 6. Unmount when done (optional)
sudo umount /mnt/nas-media
```
Same pattern for `TV-Shows` and `Anime`. Naming convention: `Movies/Title (Year)/Title (Year).mkv`.

### Content sourcing

Library is built **exclusively** from owned/ripped physical media via MakeMKV (Blu-ray/DVD) — no streaming-subscription pulls, no piracy-adjacent workflows. Note: 4K UHD ripping via MakeMKV can be less consistently reliable than standard Blu-ray due to AACS 2.0 — worth checking current compatibility before buying 4K discs in bulk.

Jellyfin was chosen over Plex specifically because Tailscale already solves remote access for free, eliminating Plex Pass's main value proposition for this use case.

---

## Tailscale — Remote Access Mesh

**Status: Live.**

- Runs as a TrueNAS app (containerized WireGuard-based mesh VPN).
- Provides secure remote access to Nextcloud and Jellyfin without any port forwarding or public exposure.
- Confirmed working across phone, desktop, laptop, and NAS.
- Also carries the Mullvad exit-node add-on — see [`privacy-vpn.md`](./privacy-vpn.md).

## Deferred / Backlog

- Bulk-populate `Movies`/`TV-Shows`/`Anime` with actual ripped content now that the ingest pipeline is verified.
- Shell alias or fstab/systemd mount unit for the SMB mount command (still manual/retyped each time).
- Optional: investigate Nextcloud's **Memories** app for better photo/video timeline browsing (may also fix desktop video playback via server-side transcoding).
- Optional: further Czkawka dedup pass between `Camera` and album folders to catch remaining phone/Takeout overlap.
