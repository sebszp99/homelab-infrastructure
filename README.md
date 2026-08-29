# Homelab Infrastructure

Self-hosted homelab: network, storage, and containerized services.
Real-world problems encountered and how I solved them.

## Architecture

- **Router**: Banana Pi BPI-R4 Pro running a custom-built OpenWrt image
- **VPN**: WireGuard for remote access
- **NAS**: Raspberry Pi 5 (4GB) + Radxa Penta SATA HAT, OpenMediaVault (OMV)
- **Storage pool**: mergerfs pool combining multiple disks, mounted at
  `/srv/mergerfs/Magazyn`
- **Virtualization**: Proxmox VE (separate hardware)
- **Backup storage**: TrueNAS (RAID1)
- **Monitoring**: Zabbix (Docker Compose on the router)
- **Services** (Docker Compose on OMV): Immich (photo/video backup), Nextcloud
- **Network extras**: IPTV bridged via VLAN, SMB/CIFS file sharing

## More

- [Deployed Services](./SERVICES.md) — what's running and how it's configured
- [Troubleshooting](./TROUBLESHOOTING.md) — real incidents and how I solved them
