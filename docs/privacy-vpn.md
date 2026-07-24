# Privacy & VPN

**Status: Mullvad live on desktop and phone via Tailscale's exit-node add-on. Laptop pending.**

## Threat Model / Layer Breakdown

| Layer | What it protects against | Tool |
|---|---|---|
| Local browsing traces | Nothing seen by ISP/sites — only local history/cookies | Incognito/private browsing |
| ISP visibility (DNS, destination, SNI, traffic metadata) | ISP logging/selling browsing activity | Commercial VPN (Mullvad) |
| Website IP-based tracking/geo-lookup | Sites identifying you by home IP | Commercial VPN (Mullvad) |
| Secure access between own devices (NAS, Proxmox, phone, laptop) | Exposing homelab services to the public internet | Tailscale (WireGuard mesh) |
| Browser fingerprinting, cross-site tracking scripts | Sites re-identifying you without cookies/IP | Brave Shields, fingerprint randomization |
| Data at rest on the NAS | Physical drive theft | ZFS native encryption |

**Key distinction:** Tailscale and Mullvad solve different problems and are complementary, not redundant. Tailscale = private device-to-device mesh. Mullvad = hiding traffic/IP from the ISP and destination sites. Tailscale's Mullvad exit-node add-on lets one client do both.

**NAS uploads specifically:** personal photo/video uploads to the NAS are already protected from ISP visibility via Tailscale's end-to-end WireGuard encryption (or pure LAN traffic when at home) — regardless of whether the NAS itself runs a Mullvad exit node. The ISP sees only encrypted traffic volume/timing, never content, filenames, or file types.

## Mullvad Rollout (via Tailscale)

Purchased the Tailscale Mullvad VPN add-on — 5 device license slots.

| Device | Platform | Exit node enabled? | Method |
|---|---|---|---|
| Desktop (`nobara-pc`) | Linux (Nobara) | ✅ Yes | CLI (`tailscale set --exit-node=...`) — no GUI tray app on this distro by default |
| Mobile (`alexanders-s25`) | Android | ✅ Yes | Tailscale app → Exit node menu |
| Laptop | Windows | ⏳ Pending | Tailscale system tray → Exit node menu (GUI-based) |
| NAS (`truenas-scale`) | TrueNAS SCALE | ❌ Deliberately not configured | See rationale below |

### Why the NAS is excluded

The NAS's role is almost entirely inbound (serving Nextcloud/Jellyfin to other tailnet devices), not outbound browsing. Its outbound traffic is limited to system updates, container image pulls, NTP, and metadata scraping — non-sensitive automated traffic. Uploads *to* the NAS are already protected by Tailscale's end-to-end encryption regardless of the NAS's own exit-node status. Enabling an exit node here risked breaking LAN reachability if `--exit-node-allow-lan-access` were misconfigured, for no meaningful privacy benefit. The third license slot was left open for a future device instead.

### Verification

```
curl ipinfo.io
```
Confirms public IP, city, region, and org — should show a Mullvad-owned network, not the home ISP. Used to verify exit-node routing on desktop and mobile.

## Additional Layering

Brave in Incognito mode alongside the Mullvad exit node closes most practical tracking vectors: ISP visibility, IP-based tracking, local history, and many ad/tracker scripts (via Brave Shields). Remaining exposure:
- Browser fingerprinting — partially mitigated by Brave's fingerprint randomization, not eliminated.
- Account-level re-identification upon logging into any service.
- Baseline trust placed in Mullvad's no-logs policy and Tailscale's own connection metadata visibility.

## Next Steps

1. Complete Tailscale + Mullvad setup on the Windows laptop.
2. Revisit a whole-home Mullvad tunnel at the OPNsense level (WireGuard client + kill-switch firewall rule) — deferred pending access point installation (see `network.md`), needed for devices that can't run a VPN client (e.g. smart TV).
3. Run leak tests (`browserleaks.com/webrtc`, `dnsleaktest.com`) to confirm no DNS/WebRTC leaks — still outstanding.

## Deferred / Backlog

- OPNsense-level Mullvad WireGuard tunnel + kill-switch firewall rule.
- Encrypted DNS (DNS-over-TLS via Unbound) at the router level.
