# Troubleshooting

Real incidents from my homelab — problem, diagnosis, and fix.

## Contents

1. [WAN port flapping, losing internet connection](#1-wan-port-flapping-losing-internet-connection)
2. [IPTV not working](#2-iptv-not-working)
3. [Immich reset to setup screen after reboot](#3-immich-reset-to-setup-screen-after-reboot-data-appeared-lost)
4. [Zabbix web UI unreachable from LAN (dual firewall)](#4-zabbix-web-ui-unreachable-from-lan-despite-working-locally-dual-firewall)
5. [OpenMediaVault SMB share read-only from iOS Files](#5-openmediavault-smb-share-writable-via-cli-read-only-from-ios-files)
6. VLAN segmentation broke all LAN/WiFi connectivity (missing PVID on bridge device)

---

## 1. WAN port flapping, losing internet connection

**Problem**: Router logs showed the WAN port going up and down repeatedly:
```
daemon.notice: netifd: Network device 'eth1' link is down
daemon.notice: netifd: Network device 'eth1' link is up
```

**Diagnosis (1)**: Checked the GPON SFP module temperature:
```sh
ethtool -m eth1
```

Module temperature: ```73.85 degrees C / 164.93 degrees F```

The module was overheating.

**Fix (1)**: Added two fans to the 10" server rack. Temperature dropped to 58°C.

**Diagnosis (2)**: A second, separate issue remained — the PPPoE session was
dropping entirely after even a few seconds of physical link loss, instead of
recovering automatically.

**Fix (2)**: Forced aggressive PPPoE keep-alive settings. In `/etc/config/network`,
WAN section:
```
option keepalive '3 5'
option lcp_echo_interval '5'
option lcp_echo_failure '3'
```

---

## 2. IPTV not working

**Setup**:
- WAN port: `eth1`
- Port for IPTV STB: `eth3`
- VLAN for ISP IPTV: `1020`
- Docker is also running on the router (relevant for troubleshooting)

**Problem**: The set-top box loaded its configuration/EPG, but showed no
picture, then later showed no picture at all — no DHCP lease.

**Diagnosis (1) — no multicast traffic on the bridge**:
```sh
tcpdump -i eth1.1020 -n udp and net 224.0.0.0/4 -c 20
```
Result: 0 packets — no video traffic arriving from the ISP side at all.

Checked the bridge's actual port membership:
```sh
bridge link show
```
Result:
```
5: eth3@eth0: ... master br-lan ...
37: eth1.1020@eth1: ... master Bridge_iptv ...
```

**Root cause**: despite adding `eth3` to `Bridge_iptv` in LuCI, the port
was still physically a member of `br-lan` (the default LAN bridge) as well.
A network port can only belong to one bridge — `eth3` was effectively
still serving LAN traffic, and `Bridge_iptv` only had one real member.

**Fix (1)**: Removed `eth3` from `br-lan`'s port list. Verified with
`bridge link show` that `eth3` now correctly showed `master Bridge_iptv`.

**Diagnosis (2) — still no DHCP after fixing bridge membership**:
```sh
tcpdump -i eth3 -n port 67 or port 68
```
Result: the STB kept sending `DHCP Discover/Request`s repeatedly, with no
response (`Offer`).

Checked whether the DHCP traffic even reached the ISP side of the bridge:
```sh
tcpdump -i eth1.1020 -n port 67 or port 68
```
Result: **no traffic** — the DHCP packets never crossed the `Bridge_iptv`
bridge, even though both ports (`eth3`, `eth1.1020`) were correctly in the
`forwarding` state.

Checked whether bridge traffic was being filtered by netfilter instead of
pure L2 switching:
```sh
sysctl net.bridge.bridge-nf-call-iptables
```
Result: `1` — traffic crossing the L2 bridge was being routed through the
firewall (`bridge-netfilter`) instead of being switched directly.

**Fix (2, test)**:
```sh
sysctl -w net.bridge.bridge-nf-call-iptables=0
```
After restarting the STB, the picture appeared — confirming the root cause.

**Fix (2, permanent)**: A plain `/etc/sysctl.d/*.conf` entry didn't survive
a reboot on its own, because the `br_netfilter` module wasn't loaded yet
when `sysctl.d` was read at boot — it loaded later, when Docker (also
running on this router) started, and Docker reset the value back to `1`.

Solution: a custom init script with a high start priority (`START=99`),
run after Docker starts:
```sh
cat > /etc/init.d/99-bridge-iptv-fix << 'EOF'
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

Verified with a full `reboot` — after restart, `sysctl
net.bridge.bridge-nf-call-iptables` correctly returned `0`, and the picture
appeared on the STB with no manual intervention.

---

## 3. Immich reset to setup screen after reboot (data appeared lost)

**Problem**: After a router-side network flapping incident caused the NAS
(Raspberry Pi 5) to fully reboot, Immich showed its welcome/setup screen
instead of the login page.

**Diagnosis**: Docker started before the `mergerfs` storage pool
(`/srv/mergerfs/Magazyn`) finished mounting, so the Immich containers
initialized against an empty local path instead of the actual data and
database.

**Fix**: Added a systemd override so `docker.service` waits for the mount
before starting:
```ini
[Unit]
RequiresMountsFor=/srv/mergerfs/Magazyn
```

Also set `restart: unless-stopped` for the `immich-server`, `redis`, and
`database` containers via OMV's `compose.override.yml` (since the main
`compose.yml` is auto-generated by OMV and gets overwritten on changes).

Verified with a full reboot test: Immich came up cleanly on the login
screen (not the welcome screen). Postgres ran WAL crash recovery
automatically on first start, with no data loss.

---

## 4. Zabbix web UI unreachable from LAN despite working locally (dual firewall)

**Setup**: Zabbix deployed via Docker Compose on an OpenWrt router
(Banana Pi BPI-R4 Pro) running both **nftables (fw4)**, the native OpenWrt
firewall, and **iptables-legacy**, managed independently by `dockerd` —
both attached to the same kernel netfilter hooks.

**Problem**: `curl -I http://localhost:18081` on the router itself returned
`200 OK`, but any browser on the LAN got an immediate **connection reset
(RST)** — too fast (~100µs) to be a real response from the container.

**Diagnosis**:

`curl` from `localhost` was misleading — it was hitting `docker-proxy`
(a process listening directly on the host), not the actual NAT/FORWARD
path that LAN traffic takes.

Traced the LAN request step by step:
```sh
tcpdump -i br-lan port 18081 -n
```
Confirmed the SYN reached the router, but the reply was an instant RST —
pointing to a router-side reject, not a network issue.

**Root cause, layer 1** — an iptables-legacy `DOCKER-USER` rule:
```
REJECT 0 -- 0.0.0.0/0 0.0.0.0/0 reject-with icmp-port-unreachable
```
This was meant to block only WAN traffic, but the init script couldn't
correctly detect the WAN interface on this platform, so it inserted the
rule with no interface filter at all — blocking everything, LAN included.

**Root cause, layer 2** — `docker compose` creates its own bridge network
with a random name (e.g. `br-acf03cfc7989`) by default. OpenWrt's native
firewall (fw4) only knows about `docker0` (added manually to UCI during
the initial Docker install). Traffic to an unrecognized bridge hit fw4's
`jump handle_reject` → RST.

**Root cause, layer 3** (the actual blocker, found after fixing 1 and 2) —
even with the new bridge properly named and added to the `docker` firewall
zone, inspecting the nftables ruleset showed the `lan` zone's forwarding
chain only had rules to reach `wan` and back to `lan` — no rule at all
forwarding `lan → docker`. The packet matched nothing and fell through to
the default reject.

**Fix**:

1. Pinned the compose network to a known bridge name:
```yaml
networks:
  zabbix-net:
    driver: bridge
    driver_opts:
      com.docker.network.bridge.name: br-zabbix
```

2. Added that bridge to the `docker` firewall zone in UCI:
```sh
uci add_list firewall.docker.device='br-zabbix'
uci commit firewall
service firewall reload
```

3. Added the missing forwarding rule:
```sh
uci add firewall forwarding
uci set firewall.@forwarding[-1].src='lan'
uci set firewall.@forwarding[-1].dest='docker'
uci commit firewall
service firewall reload
```

4. Re-secured WAN access explicitly (defense-in-depth, since the original
`DOCKER-USER` blanket-block rule had been removed during diagnosis):
```sh
uci add firewall rule
uci set firewall.@rule[-1].name='Block-WAN-to-Docker'
uci set firewall.@rule[-1].src='wan'
uci set firewall.@rule[-1].dest='docker'
uci set firewall.@rule[-1].target='REJECT'
uci set firewall.@rule[-1].family='any'
uci commit firewall
service firewall reload
```

Verified from an external network (mobile data, not Wi-Fi) that the
web UI is unreachable from WAN.

**Takeaway**: on this platform, two independent firewalls sit on the same
netfilter hooks. An instant RST almost always means a REJECT in one of
them — worth checking both `iptables -L` and `nft list ruleset`
separately, since they're entirely independent rule sets.

---

## 5. OpenMediaVault SMB share writable via CLI, read-only from iOS Files

**Setup**: OMV on Raspberry Pi, SMB share `Pi_NAS` on a mergerfs volume,
accessed from an iPhone via the Files app over SMB 3.11.

**Problem**: The share was visible and browsable from iOS, but "New
Folder" was greyed out and no files could be written — despite logging in
with a real (non-guest) account.

**Diagnosis — ruling out permissions**: `getfacl` confirmed `rwx` for the
correct group, and `sudo -u <user> touch file` succeeded — filesystem
permissions were fine. `smb.conf` had `read only = no` and the correct
write list. `smbstatus -b` confirmed the session authenticated correctly
as the real user over SMB3. Writing directly via `smbclient` from the
command line also succeeded — meaning the raw SMB protocol path allowed
writes, but the iOS Files client specifically did not.

**Root cause**: `smb.conf` had the Apple-compatibility VFS module enabled
globally:
```ini
[global]
vfs objects = fruit streams_xattr
```
But the specific `[Pi_NAS]` share had this same parameter explicitly
overridden to empty:
```ini
[Pi_NAS]
vfs objects =
```
In Samba, list-type parameters like `vfs objects` are fully overridden at
the share level, not merged with the global config — so `fruit` was
silently disabled for this one share. iOS relies on Apple's protocol
extensions to correctly interpret permissions and file attributes, and
without them it defaulted to treating the share as read-only, even though
the underlying permissions were correct.

**Fix**: In OMV's share config ("Extra options" field), added, on
separate lines:
```ini
vfs objects = fruit streams_xattr
fruit:metadata = stream
```
(combining both directives on one line causes Samba to misparse
everything after the first `=` as a single value). Restarted the service:
```sh
sudo systemctl restart smbd
```
After reconnecting on iPhone, writes worked correctly.

**Takeaway**: "read-only, but only from iOS/macOS, while other SMB clients
work fine" is a strong signal to check the `fruit` VFS module
configuration specifically, rather than ACL/POSIX permissions.

---

## 6. VLAN segmentation broke all LAN/WiFi connectivity (missing PVID on bridge device)

**Setup**: Implementing network segmentation on the router — VLAN 1 (trusted),
VLAN 20 (IoT), VLAN 30 (guest) — using OpenWrt's DSA-based Bridge VLAN
filtering on `br-lan` (MaxLinear MxL862XX switch chip, no `swconfig`). VLAN 1
untagged on the trusted ports, VLAN 20/30 tagged on the trunk port to the
Cudy AP.

**Problem**: After `Save & Apply` in LuCI, every client on the LAN — wired
and Wi-Fi — lost connectivity. Wired clients got `Destination host
unreachable` when pinging the router (a locally-generated error, meaning ARP
never resolved). Wi-Fi SSIDs were still visible and clients could associate,
but never received a DHCP lease. The issue survived multiple full
power-cycles, ruling out "unsaved changes, waiting for LuCI's rollback timer"
— that timer does not survive a reboot.

**Diagnosis**: Recovered access out-of-band via the router's USB-C serial
console (115200 baud), since no network path was available.

Confirmed the UCI config was intact and correctly described the intended
VLANs:

```
uci show network
```

Ruled out a bad physical link by testing on a different port — same
failure.

Confirmed `dnsmasq` was running with correct `dhcp-range` entries for `lan`,
`iot`, and `guest`:

```
ps | grep dnsmasq
cat /var/etc/dnsmasq.conf.cfg01411c
```

Captured live traffic directly on the bridge:

```
tcpdump -i br-lan -n port 67 or port 68
```

Result: `DHCPREQUEST` packets from clients **were** arriving at `br-lan`,
but the router never responded.

Checked the firewall for anything blocking UDP 67/68:

```
nft list ruleset
```

No blocking rule existed — `accept_from_lan`/`accept_to_lan` were correctly
permissive.

Inspected the switch chip's actual hardware VLAN table (not just the
Linux-side UCI config) — this was the key step:

```
bridge vlan show
```

Every physical port showed `1 PVID Egress Untagged`, except the bridge
device itself:

```
mxl_lan3          1 PVID Egress Untagged
                  20
                  30
br-lan            1              <-- missing "PVID Egress Untagged"
                  20
                  30
```

**Root cause**: `br-lan` isn't just a Layer 2 forwarding device — it's also
the interface the router's own IP stack (and `dnsmasq`) listens on.
Untagged VLAN 1 frames were correctly switched in hardware onto the bridge,
but because the bridge's own entry in the kernel VLAN table lacked the PVID
(default VLAN) and untagged (egress) flags, those frames were never handed
up to the host's IP stack. `dnsmasq` never actually saw the DHCP requests
at the application layer, even though `tcpdump` showed them on the wire —
that capture point sits below where this filtering happens.

Critically, setting `local='1'` on the `bridge-vlan` UCI section —

```
uci set network.@bridge-vlan[0].local='1'
```

— adds the bridge as a VLAN *member*, but does not automatically set the
PVID/untagged flags for it on this driver combination (MaxLinear MxL862XX +
DSA). This looks like a driver-level gap rather than intended OpenWrt
behavior.

**Fix (test)**: Applied directly to the kernel's bridge VLAN table:

```
bridge vlan add vid 1 dev br-lan self pvid untagged
```

Connectivity was restored instantly.

**Fix (permanent)**: This setting isn't stored in UCI and doesn't survive
`/etc/init.d/network restart` or a reboot, so a hotplug script re-applies it
every time the `lan` interface comes up:

```
cat > /etc/hotplug.d/iface/95-fix-lan-vlan1 << 'EOF'
#!/bin/sh
[ "$ACTION" = "ifup" ] || exit 0
[ "$INTERFACE" = "lan" ] || exit 0
bridge vlan add vid 1 dev br-lan self pvid untagged 2>/dev/null
EOF
chmod +x /etc/hotplug.d/iface/95-fix-lan-vlan1
```

Verified with both `/etc/init.d/network restart` and a full reboot —
`bridge vlan show` correctly shows `br-lan 1 PVID Egress Untagged` after
each, with no manual intervention.

**Takeaway**: When VLAN filtering misbehaves on DSA, `bridge vlan show` is
the ground truth — `uci show network` only describes intent, not what the
hardware is actually enforcing. The bridge device itself needs a PVID, not
just its member ports, and that's easy to miss since most guides only cover
per-port tagged/untagged settings. Also: `tcpdump` showing traffic on an
interface is not proof a service is receiving it — frames can be visible on
the wire while still being dropped before reaching the application layer.
Keeping a serial console fallback confirmed *before* touching the LAN
bridge turned a total lockout into a 2-hour fix instead of a full
reflash.
