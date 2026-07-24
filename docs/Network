# Network

**Status: Core routing/switching operational. WiFi gap identified, fix in progress.**

## Topology

```
Internet
   │
Verizon Gateway  (double-NAT; currently also broadcasts household WiFi)
   │  Port 1 — WAN uplink (VLAN 10)
Netgear GS308E (managed switch)
   │  Port 2 — OPNsense trunk (VLAN 1, 10, 20)
OPNsense (router/firewall, DHCPv4)
   │  Default gateway: 192.168.1.1
   │  Ports 3–8 — VLAN 20 (LAN), untagged access, DHCP 192.168.1.x
   ├── Desktop PC (Linux/Nobara) — DHCP lease
   ├── Proxmox VE — static IP on vmbr0 (by design, won't show in DHCP leases)
   └── TrueNAS SCALE — DHCP lease within VLAN 20
```

## Router / Firewall — OPNsense

- Runs headlessly, handles DHCPv4 and routing.
- Sits behind the Verizon gateway in a stable double-NAT arrangement.
- Default gateway: `192.168.1.1`.
- No outstanding configuration work on the routing layer itself.

## Switching — Netgear GS308E (managed)

| Port | Connected Device | VLAN | PVID | Role |
|---|---|---|---|---|
| 1 | Verizon Router / Internet Uplink | VLAN 10 (WAN) | 10 | Feeds external internet into the switch |
| 2 | OPNsense (em0) | VLAN 1, 10, 20 (Trunk) | 1 | Main trunk line to/from the router |
| 3–8 | LAN devices | VLAN 20 (LAN) | 20 | Any device here gets a `192.168.1.x` IP |

## Switching — Netgear JGS524 (unmanaged, 24-port)

- Confirmed plain/unmanaged, non-PoE variant.
- Cannot handle VLAN duties (that stays with the GS308E) — will act purely as a downstream port expander.
- **Plan:** connect to an open VLAN 20 port on the GS308E once it fills up (expected once the access point is installed).
- **Status: not yet installed.**

## WiFi — Gap Identified & Fix In Progress

**The problem:** household WiFi is currently broadcast by the Verizon gateway, which sits *upstream* of OPNsense. This means WiFi-connected devices (phone, etc.) bypass OPNsense entirely — no VLAN policy, firewall rule, or future router-level VPN tunnel applies to them.

**The fix:** a dedicated wireless **access point** (not a router) connected to an open GS308E LAN port (VLAN 20). An AP purely bridges wireless clients onto the existing OPNsense-managed LAN — it doesn't route, NAT, or hand out DHCP. Once live, WiFi devices become normal downstream LAN clients of OPNsense, and the Verizon gateway's own WiFi radios should be disabled.

### Access point evaluation

| Model | Verdict | Key factor |
|---|---|---|
| Cisco Aironet 3802I | Rejected | Ships in Lightweight mode requiring a Cisco WLC; reflash requires console cable, TFTP, and a Cisco support login |
| Ruckus H510 | Rejected for this use case | Wall-plate, 2x2 MIMO — built for single-room, not whole-house |
| TP-Link Omada EAP670 | Viable alternative | Free controller, ships with its own power adapter |
| Ubiquiti U6 Pro / U6 Lite | Viable alternative | Free self-hosted UniFi controller (could run as an LXC on Proxmox); U6 Pro better for whole-house range |
| **Ruckus R710 (901-R710-US00)** | **Selected — purchased via eBay** | Enterprise 4x4:4 MU-MIMO with BeamFlex+; strong price-to-performance used |
| Ruckus R720 (901-R720-US00) | Considered, not selected | Better spec (2.5GbE) but not needed on a gigabit-only network |

**Firmware:** the R710 does not need to ship pre-loaded as "Unleashed" (9U1- SKU) — Ruckus provides a documented process to convert a standard image via local web UI: factory reset → set PC to a `192.168.0.x` static IP → log in at `192.168.0.1` (`super`/`sp-admin`) → Maintenance → Upgrade → Local → factory reset again. No console cable or TFTP server required.

> For future AP purchases: look for the "9U1-" SKU prefix for guaranteed pre-loaded Unleashed firmware.

### PoE

The GS308E has no PoE ports and the R710 requires 802.3af PoE, so a separate injector was required. **Selected:** a gigabit-rated (10/100/1000Mbps), 802.3af/at-compliant injector, 48V output.

**Status: AP and PoE injector ordered, not yet arrived. Installation and any Unleashed conversion deferred until hardware arrives.**

## Next Steps

1. Receive Ruckus R710 and PoE injector; confirm firmware version.
2. Convert to Unleashed via local web UI if not already pre-loaded.
3. Connect R710 to an open GS308E port (VLAN 20); confirm it pulls a DHCP lease from OPNsense.
4. Disable Verizon gateway WiFi radios once the R710's SSID is live and confirmed working.
5. Install the JGS524 switch once GS308E port capacity is reached.
6. Revisit a whole-home Mullvad WireGuard tunnel + kill-switch at the OPNsense level (for devices that can't run a VPN client, e.g. the smart TV) — deferred pending AP installation.
7. Add encrypted DNS (DNS-over-TLS via Unbound) at the router level.
8. Run leak tests (`browserleaks.com/webrtc`, `dnsleaktest.com`) to confirm no DNS/WebRTC leaks — still outstanding.
