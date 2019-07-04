# OpenWRT VPN Setup

## Install openvpn

```
# opkg update
# opkg install openvpn-openssl
# service openvpn enable
```

## Configure new vpn

Reference: [OpenVPN reference](https://openvpn.net/community-resources/reference-manual-for-openvpn-2-4/)
Guides:
 * http://www.linuxhorizon.ro/iproute2.html
 * https://openwrt.org/docs/guide-user/services/vpn/openvpn/client
 * https://serverfault.com/questions/144048/linux-port-based-routing-using-iptables-ip-route
 * https://medium.com/@ingamedeo/openvpn-splittunneling-on-openwrt-e4302a1a4e12

### Prepare

Add openvpn configuration to `/etc/openvpn/<name>/`.

```
# mkdir /etc/openvpn/<name>
# touch /etc/openvpn/<name>/config.ovpn
# chmod 600 /etc/openvpn/<name>/config.ovpn
# touch /etc/openvpn/<name>/route-up.sh
# chmod 700 /etc/openvpn/<name>/route-up.sh
# touch /etc/openvpn/<name>/route-down.sh
# chmod 700 /etc/openvpn/<name>/route-down.sh
# touch /etc/openvpn/<name>/auth.txt
# chmod 600 /etc/openvpn/<name>/auth.txt
```

### Client config

Add the following to `/etc/openvpn/<name>/config.ovpn`:

```
auth-user-pass auth.txt
# Dropping privileges prevents down script from removing routes
# user nobody
# group nogroup
script-security 2
route-noexec
route-up route-up.sh
route-pre-down route-down.sh
```

Change the following in `/etc/openvpn/<name>/config.ovpn`:

```
- dev tun
+ dev <name>vpn
+ dev-type tun
```

### Authentication

Add your username and password to `/etc/openvpn/<name>/auth.txt`:

```
# echo '<USERNAME>\n<PASSWORD>' > /etc/openvpn/<name>/auth.txt
```

### IP Routing

Example for `/etc/openvpn/<name>/route-up.sh`:

```
#!/bin/sh

echo "UP: $dev : $ifconfig_local -> $ifconfig_remote gw: $route_vpn_gateway" >> routes.log

# Set the default route to use the vpn table
/sbin/ip route add ${route_network_1}/${route_netmask_1} via ${route_gateway_1} dev ${dev}
/sbin/ip route add default via ${route_vpn_gateway} dev ${dev} table ${dev}

# This needs to be a unique number for every vpn. See /etc/iproute2/rt_tables for existing values
UNIQUE_MARK=2

# Add vpn iptable
/bin/grep -qxF "${UNIQUE_MARK} ${dev}" || /bin/echo "${UNIQUE_MARK} ${dev}" >> /etc/iproute2/rt_tables

# Mark all packets that should be routed through the vpn
# This iptables rule will mark all packets from lan with destination port 80
/usr/sbin/iptables -A PREROUTING -t mangle -i br-lan -p tcp --dport 80 -j MARK --set-mark ${UNIQUE_MARK}

# Route all marked packets through the vpn
/sbin/ip rule add from all fwmark ${UNIQUE_MARK} table ${dev}
```

Example for `/etc/openvpn/<name>/route-down.sh`:

```
#!/bin/sh

echo "DOWN: $dev : $ifconfig_local -> $ifconfig_remote gw: $route_vpn_gateway" >> routes.log

# Set the default route to use the vpn table
/sbin/ip route del ${route_network_1}/${route_netmask_1} via ${route_gateway_1} dev ${dev}
/sbin/ip route del default via ${route_vpn_gateway} dev ${dev} table ${dev}

# This needs to be a unique number for every vpn. See /etc/iproute2/rt_tables for existing values
UNIQUE_MARK=2

# TODO remove table?
# Add the vpn ip table
# /bin/grep -qxF "${UNIQUE_MARK} ${dev}" || /bin/echo "${UNIQUE_MARK} ${dev}" >> /etc/iproute2/rt_tables

# Mark all packets that should be routed through the vpn
# This iptables rule will mark all packets from lan with destination port 80
/usr/sbin/iptables -D PREROUTING -t mangle -i br-lan -p tcp --dport 80 -j MARK --set-mark ${UNIQUE_MARK}

# Route all marked packets through the vpn
/sbin/ip rule del from all fwmark ${UNIQUE_MARK} table ${dev}
```

### Add firewall zone

Add the following to `/etc/config/firewall`:

```
config zone
        option name '<name>vpn'
        option forward 'REJECT'
        option output 'ACCEPT'
        option input 'REJECT'
        option masq '1'
        option mtu_fix '1'
        option network '<name>vpn'

config forwarding
        option dest '<name>vpn'
        option src 'lan'
```

### Add openvpn entry

Add the following to `/etc/config/openvpn`:

```
config openvpn <name>
        option enabled 1
        option config /etc/openvpn/<name>/config.ovpn
```

### Add network interface

Add the following to `/etc/config/network`:

```
config interface '<name>vpn'
        option ifname '<name>vpn'
        option proto 'none'
        option auto '1'
```
