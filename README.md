# MAP-E for OpenWrt and GL.iNet routers

A replacement for OpenWrt's `/lib/netifd/proto/map.sh` that makes MAP-E
(IPv4 over IPv6) work properly with Japanese IPoE services such as those
running over NTT FLET'S, where many customers share one IPv4 address and
each one gets a set of port ranges.

Tested on a GL.iNet [GL-MT6000 (Flint 2)](https://openwrt.org/toh/gl.inet/gl-mt6000)
running GL.iNet firmware 4.11.0 (OpenWrt 21.02 base, kernel 5.4.238).
It is derived from a script that was tested on stock OpenWrt 22.03 and 23.05
(see [Credits](#credits)).

## What it changes compared to the stock map.sh

1. **SNAT to all assigned port ranges.** The stock script ends up using only
   the first block of 16 ports, so pages that open many IPv4 connections
   stall and IPv4 ping does not work. This version creates an nftables table
   `inet mape` that spreads connections over every assigned port with
   `numgen inc`.
2. **Clean restart.** The `inet mape` table is deleted and rebuilt each time
   the interface comes up, so `ifup` never stacks duplicate rules.
3. **Tunnel MTU 1460 instead of 1280.** 1460 is the largest that fits in a
   1500-byte IPv6 link (40 bytes go to the IPv6 header). 1280 is too small
   for QUIC-based VPNs such as Cloudflare WARP.
4. **MediaTek hardware NAT in MAP-E mode.** GL.iNet firmware for MediaTek
   SoCs ships the `mtkhnat` driver, which treats IPv4-in-IPv6 tunnels as
   DS-Lite by default. In that mode forwarded IPv4 connections stall after
   roughly 100 KB. The script switches it to MAP-E mode at setup time. On
   routers without `/sys/kernel/debug/hnat/mape_toggle` this step is skipped.

You can set a different MTU with `option mtu` on the MAP-E interface.

## Requirements

- The `map` package.
- `kmod-nft-nat`, which provides the `nft_chain_nat` kernel module. Without
  it the script logs `Error: Could not process rule: No such file or
  directory` and IPv4 comes up without working SNAT. Recent GL.iNet
  firmware (4.11.0) no longer includes it.
- The `nft` command. GL.iNet 4.x firmware and OpenWrt 22.03 or later
  already have it. On plain OpenWrt 21.02, install `nftables` too.

## Installation

From a computer on the router's LAN (`192.168.8.1` is the GL.iNet default):

```sh
# Keep a copy of the original script
ssh root@192.168.8.1 'cat /lib/netifd/proto/map.sh' >map.sh.orig

# Install packages
ssh root@192.168.8.1 'opkg update && opkg install map kmod-nft-nat'

# Install the script
cat map.sh | ssh root@192.168.8.1 'cat >/lib/netifd/proto/map.sh'
ssh root@192.168.8.1 'chmod +x /lib/netifd/proto/map.sh'
```

Install the packages first, so that installing `map` cannot replace the
script afterwards. On OpenWrt releases that use `apk` instead of `opkg`, use
`apk update && apk add map kmod-nft-nat`.

Then reboot the router, or restart the MAP-E interface with
`ifup WAN6MAPE` (use your interface name).

**After a firmware upgrade**, check that both packages are still installed
and that `/lib/netifd/proto/map.sh` is still this version. A firmware
upgrade can replace either one.

## Configuration

The IPv6 side (WAN6 with DHCPv6, relay mode on LAN, and so on) and the MAP-E
parameters depend on your ISP. fakemanhk's
[openwrt-jp-ipoe guide](https://github.com/fakemanhk/openwrt-jp-ipoe)
walks through the whole LuCI setup, and the
[online MAP-E rule calculator](http://ipv4.web.fc2.com/map-e.html) turns
your WAN6 IPv6 address into the values below.

The resulting interface in `/etc/config/network` looks like this:

```
config interface 'WAN6MAPE'
	option proto 'map'
	option maptype 'map-e'
	option peeraddr '<BR address>'
	option ipaddr '<IPv4 prefix>'
	option ip4prefixlen '<IPv4 prefix length>'
	option ip6prefix '<IPv6 prefix>'
	option ip6prefixlen '<IPv6 prefix length>'
	option ealen '<EA-bits length>'
	option psidlen '<PSID length>'
	option offset '<PSID offset>'
	option tunlink 'wan6'
	option legacymap '1'
```

Put the MAP-E interface in the same firewall zone as `wan` and `wan6`.

If you run servers on some of your assigned ports, list them in
`DONT_SNAT_TO` near the top of `map.sh` so outgoing connections do not use
them.

## Checking that it works

On the router:

```sh
# Tunnel up with the expected MTU and your shared IPv4 address
ip addr show map-WAN6MAPE

# One SNAT rule each for icmp, tcp and udp
nft list chain inet mape srcnat

# GL.iNet / MediaTek only: should print "mape is enabled now!"
cat /sys/kernel/debug/hnat/mape_toggle; dmesg | tail -1
```

From a LAN client, an IPv4 download should run at full speed:

```sh
curl -4 -o /dev/null https://speed.cloudflare.com/__down?bytes=20000000
```

## Troubleshooting

| Symptom | Cause |
|---|---|
| `Could not process rule: No such file or directory` in `logread` | `kmod-nft-nat` is not installed |
| IPv4 downloads from LAN clients stall after ~100 KB, but downloads made on the router itself work; IPv6 is fine | MediaTek hardware NAT is in DS-Lite mode; check `mape_toggle` as above |
| QUIC-based VPNs fail over IPv4 | Tunnel MTU too small; check `ip link show map-WAN6MAPE` |
| `ip6_tunnel: map-WAN6MAPE xmit: Local address not yet configured!` at boot | Harmless; the tunnel starts before its IPv6 address is ready |

## Credits

- Original `map.sh`: Steven Barth, from OpenWrt's `map` package
  (Copyright (c) 2014 cisco Systems, Inc., GPL-2.0).
- SNAT over all port ranges and the clean table restart: cmspam
  ([cmspam/openwrt-jp-ipoe](https://github.com/cmspam/openwrt-jp-ipoe)).
- Setup guide and the published modified script: fakemanhk
  ([fakemanhk/openwrt-jp-ipoe](https://github.com/fakemanhk/openwrt-jp-ipoe)).
- MAP-E rule calculator: [ipv4.web.fc2.com/map-e.html](http://ipv4.web.fc2.com/map-e.html).
- MTU and MediaTek hardware NAT fixes: Bernie Innocenti.

## License

GPL-2.0, like the OpenWrt script it is derived from. See [LICENSE](LICENSE).
