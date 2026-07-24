# CCNA Lab Environment

**Status: Not started — next major project.**

## Goal

Use the Proxmox node to run a virtual lab environment for CCNA study: routing, switching, and network simulation practice, separate from the "production" homelab services running elsewhere on the same host.

## Planning notes

*(To be filled in as decisions are made — placeholder for now.)*

- Topology tool: TBD (e.g. GNS3, EVE-NG, or Cisco Modeling Labs running as a VM on Proxmox).
- Resource allocation: how much of the Proxmox node's CPU/RAM/local NVMe to reserve for lab VMs vs. existing/future production workloads.
- Networking: whether lab traffic should be isolated on its own VLAN so simulated router/switch misconfigurations can't affect the real LAN.
- Image sourcing: which Cisco IOS/IOU images or vendor images are needed and how they'll be obtained legitimately.
- Persistence: whether lab configs/topologies get backed up (e.g. to `hdd-pool/lab-backups`, already provisioned — see `storage.md`).

## Next Steps

1. Decide on the lab platform (GNS3 vs EVE-NG vs CML) and confirm it will run acceptably on the existing Proxmox hardware.
2. Decide on VLAN/network isolation strategy so lab traffic can't leak into the production LAN.
3. Stand up the first VM/container and confirm image imaging works.
4. Document the working setup here and log progress in `CHANGELOG.md`.
