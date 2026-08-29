# Homelab Infrastructure

Self-hosted homelab: network, storage, and containerized services. 
Real-world problems encountered and how I solved them.

## Architecture

- **Router**: Banana Pi BPI-R4 Pro running a custom-built OpenWrt image
- **VPN**: WireGuard for remote access
- **NAS**: Raspberry Pi 5 (4GB) + Radxa Penta SATA HAT, OpenMediaVault (OMV)
- **Storage pool**: mergerfs pool combining multiple disks, mounted at 
  /srv/mergerfs/Magazyn
- **Virtualization**: Proxmox VE (separate hardware)
- **Backup storage**: TrueNAS (RAID1)
- **Services** (Docker Compose on OMV): Immich (photo/video backup), 
  Jellyfin (media streaming), Nextcloud

## More

- [Deployed Services](./SERVICES.md) — what's running and how it's configured
- [Troubleshooting](./TROUBLESHOOTING.md) — real incidents and how I solved them
