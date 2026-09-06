# Homelab Infrastructure

Self-hosted homelab: network, storage, and containerized services.
Real-world problems encountered and how I solved them.

## Architecture

- **Router**: Banana Pi BPI-R4 Pro running a custom-built OpenWrt image
- **Network segmentation**: VLANs (trusted / IoT / guest) with per-zone
firewall isolation between them
- **DNS filtering**: AdGuard Home (network-wide ad/tracker/malware blocking
across all VLANs)
- **VPN**: WireGuard for remote access
- **NAS**: Raspberry Pi 5 (4GB) + Radxa Penta SATA HAT, OpenMediaVault (OMV)
- **Storage pool**: mergerfs pool combining multiple disks, mounted at
/srv/mergerfs/Magazyn
- **Virtualization**: Proxmox VE (separate hardware)
- **Backup storage**: TrueNAS (RAID1)
- **Services** (Docker Compose on OMV): Immich (photo/video backup),
Jellyfin (media streaming), Nextcloud

## More

- [Deployed Services](https://github.com/sebszp99/homelab-infrastructure/blob/main/SERVICES.md) — what's running and how it's configured
- [Troubleshooting](https://github.com/sebszp99/homelab-infrastructure/blob/main/TROUBLESHOOTING.md) — real incidents and how I solved them
