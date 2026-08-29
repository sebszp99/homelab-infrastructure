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

## Problems & Solutions

### 1. WAN port flaping, losing internet connection
**Problem**: Router logs showed problem with wan port eth1 

daemon.notice: netifd: Network device 'eth1' link is down
daemon.notice: netifd: Network device 'eth1' link is up

**Diagnosis**: First problem was temperature on gpon stick

ethtool -m eth1
Module temperature                        : 73.85 degrees C / 164.93 degrees F

**Fix**: Added two fans to my Lab Rax 10" Server Rack, the temperature dropped to 58 degrees C.
**Diagnosis**: Second problem was wich losing pppoe connection after physical link droped (even it was few seconds)
**Fix**: Forcing aggresive Keep-Alive setting for PPPoE. In /etc/config/network in WAN section added new parameters:

option keepalive '3 5'
option lcp_echo_interval '5'
option lcp_echo_failure '3'

### 2. IPTV not working
- WAN port: `eth1`
- Port for IPTV STB: `eth3`
- VLAN for (ISP) IPTV: `1020`
- Docker is also running on the router (relevant for troubleshooting)
**Problem**: The set-top box loaded the configuration/EPG, but there is no picture.
**Dioagnosis**:
  ```sh
tcpdump -i eth3 -n port 67 or port 68
```
Result: The set-top box kept sending `DHCP Request`s over and over, with no response (`Offer`).

Checking whether the request even passes through the bridge to the service provider's side:
```sh
tcpdump -i eth1.1020 -n port 67 or port 68
```
Result: **no traffic** — the DHCP packet did not pass through the `Bridge_iptv` bridge, even though both ports (`eth3`, `eth1.1020`) were correctly in the `forwarding` state.

Diagnosis — Filtering bridge traffic via netfilter

```sh
sysctl net.bridge.bridge-nf-call-iptables
```
Result: `1` — this means that traffic passing through the L2 bridge was, after all, being routed through the firewall (`bridge-netfilter`) instead of being pure switching.

### Solution (Part 2) — Disabling bridge-netfilter

Temporary test:
```sh
sysctl -w net.bridge.bridge-nf-call-iptables=0
```
After restarting the set-top box, the picture reappeared — confirming the source of the problem.

### Making the Change Permanent After Restarting the Router

The standard `/etc/sysctl.d/*.conf` file **did not work on its own**, because the `br_netfilter` module had not yet been loaded when `sysctl.d` was read at system startup—it loaded later, when the **Docker** service (present on this router) started, which overrode the value back to `1`.

Solution: a custom startup script with a very high startup number (`START=99`), executed after Docker starts:

```sh
cat > /etc/init.d/99-bridge-iptv-fix << ‘EOF’
#!/bin/sh /etc/rc.common
START=99
STOP=01

start() {
    sysctl -w net.bridge.bridge-nf-call-iptables=0
    sysctl -w net.bridge.bridge-nf-call-ip6tables=0
}
EOF

chmod +x /etc/init.d/99-bridge-iptv-fix
/etc/init.d/99-bridge-iptv-fix enable
```

Verified with a full `reboot`—after the restart, `sysctl net.bridge.bridge-nf-call-iptables` correctly returned `0`, and the picture appeared on the set-top box without any manual intervention.



