# Deployed Services

What's running in the homelab and how it's configured.

## Monitoring — Zabbix

Deployed via Docker Compose on the Banana Pi BPI-R4 Pro router (NVMe storage,
Docker Root Dir relocated from the default overlay partition to NVMe via UCI —
editing `/etc/docker/daemon.json` directly has no effect on this platform,
since `dockerd` regenerates its config from UCI on every start).

**Stack**: `mysql:8.0`, `zabbix/zabbix-server-mysql`, `zabbix/zabbix-web-nginx-mysql`,
on a dedicated bridge network (`br-zabbix`), web UI on port `18081`.

**Monitored hosts**:
- The router itself (native `zabbix-agentd` via `apk`)
- The RPi5/OMV NAS (`zabbix-agent2`), including a custom CPU temperature
  metric added via `UserParameter` reading `/sys/class/thermal/thermal_zone0/temp`
- A Wi-Fi access point with no SNMP support in its firmware, monitored via
  ICMP ping (availability/latency only)

**Security**: LAN access to the dashboard is intentionally allowed
(`lan → docker` firewall forwarding); direct WAN access is explicitly
rejected with a dedicated firewall rule, verified from an external network.

See [Troubleshooting](./TROUBLESHOOTING.md) for the firewall issue this
setup surfaced.

## Photo & Video Backup — Immich

Docker Compose stack on the RPi5/OMV NAS: `immich-server`, `redis`
(`valkey/valkey`), and Postgres with the vector extension
(`ghcr.io/immich-app/postgres`). The `immich-machine-learning` container
(face/object recognition) was deliberately left out — not worth the
resource cost on a 4GB RAM, GPU-less Raspberry Pi for this use case.

Automatic backup from iPhone via the Immich app. A read-only external
library ("Pi NAS — Archive") gives access to ~57,000 older photos/videos
migrated from a separate TrueNAS/Nextcloud system.

Chosen over Jellyfin (couldn't handle iPhone's HEIC photo format — video-only
tool) and over PhotoPrism (Immich has a proper iOS app with background
auto-backup).

See [Troubleshooting](https://github.com/sebszp99/homelab-infrastructure/blob/main/TROUBLESHOOTING.md#3-immich-reset-to-setup-screen-after-reboot-data-appeared-lost) for the database recovery
incident.

## IPTV — VLAN bridging on OpenWrt

The ISP's IPTV VLAN (`1020`) is bridged transparently from the WAN port to
a dedicated port for the set-top box, so the STB behaves as if connected
directly to the ISP's network — no routing or NAT.

**Config**: a VLAN sub-interface (`eth1.1020`) bridged with the STB port,
bridge VLAN filtering disabled (plain L2 bridge), IGMP snooping + multicast
querier enabled on the bridge, interface assigned to the `wan` firewall zone.

See [Troubleshooting](https://github.com/sebszp99/homelab-infrastructure/blob/main/TROUBLESHOOTING.md#2-iptv-not-working) for two issues this required
fixing: a bridge port membership conflict, and a bridge-netfilter DHCP block.

## File Sharing — SMB/CIFS (OpenMediaVault)

SMB shares on the RPi5/OMV NAS, backed by a mergerfs pool combining
multiple disks. Apple-client compatibility (`vfs objects = fruit
streams_xattr`) enabled globally for correct permission/attribute
handling from iOS/macOS clients.

See [Troubleshooting](./TROUBLESHOOTING.md) for a per-share config override
that silently broke this for one share.
