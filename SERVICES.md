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

See [Troubleshooting](https://github.com/sebszp99/homelab-infrastructure/blob/main/TROUBLESHOOTING.md#4-zabbix-web-ui-unreachable-from-lan-despite-working-locally-dual-firewall) for the firewall issue this
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

See [Troubleshooting](https://github.com/sebszp99/homelab-infrastructure/blob/main/TROUBLESHOOTING.md#5-openmediavault-smb-share-writable-via-cli-read-only-from-ios-files) for a per-share config override
that silently broke this for one share.

## Network Segmentation — VLANs (OpenWrt)

Three isolated networks on the router (Banana Pi BPI-R4 Pro, DSA-based
Bridge VLAN filtering, MaxLinear MxL862XX switch chip): `lan` (trusted,
VLAN 1), `iot` (VLAN 20), and `guest` (VLAN 30). The Cudy AP carries all
three over a single trunk port to the router (`Banana` / `Banana_IoT` /
`Banana_guest` SSIDs, VLAN 1 untagged + 20/30 tagged).

**Config**: `iot` and `guest` are separate firewall zones with default-deny
forwarding to `lan`; both are allowed to reach `wan` (internet) but nothing
else. One explicit exception: a `lan`+`WireGuard_VPN` → `iot` traffic rule,
so IoT devices can be managed both locally and remotely over VPN, while the
reverse direction stays blocked.

**Hardening rationale**: unused physical switch ports are left out of every
VLAN entirely (not even VLAN 1), so plugging a device into a spare port
doesn't grant it network access by default.

See [Troubleshooting](https://github.com/sebszp99/homelab-infrastructure/blob/main/TROUBLESHOOTING.md#6-vlan-segmentation-broke-all-lanwifi-connectivity-missing-pvid-on-bridge-device)
for two issues this setup required fixing: a missing PVID flag on the
bridge device itself (killed all LAN/Wi-Fi connectivity), and
[missing DHCP/DNS firewall rules](https://github.com/sebszp99/homelab-infrastructure/blob/main/TROUBLESHOOTING.md#7-custom-firewall-zones-iotguest-associated-to-wi-fi-but-never-got-a-dhcp-lease)
for the two new custom zones.

## DNS Filtering — AdGuard Home

Network-wide ad/tracker/malware blocking, deployed as a Docker container on
the router (same NVMe-backed Docker setup used for Zabbix), replacing
`dnsmasq`'s DNS role across all three VLANs (`lan`/`iot`/`guest`) at once.

**Stack**: `adguard/adguardhome`, Docker Compose,
`/mnt/nvme0n1p3/adguardhome`, `network_mode: host` (needed so a single
instance can serve DNS on the router's LAN, IoT, and guest interfaces
simultaneously without per-VLAN port mapping).

**Config**: `dnsmasq` reconfigured to DHCP-only (`dhcp.@dnsmasq[0].port='0'`)
to free port 53 for AdGuard Home. Upstream resolver is Quad9 over DoH.
Admin UI moved to port `3001` (port 80/3000 conflicted with LuCI/uhttpd).
Blocklists: AdGuard DNS filter, OISD Blocklist Big, Steven Black's List.
No DHCP changes were needed on the client side — clients already used the
router's own IP as DNS, which AdGuard Home now answers on instead of
`dnsmasq`.

**Result**: ~25-30% of DNS queries blocked in normal daily use (ad/tracker
domains), transparently across the trusted, IoT, and guest networks.
