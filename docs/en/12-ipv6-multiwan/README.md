# Chapter 12 — IPv6 Multi-WAN

> Dual-stack and IPv6-only Multi-WAN on MikroTik RouterOS 7.

---

## 12.1 IPv6 Multi-WAN Differences from IPv4

| Aspect | IPv4 | IPv6 |
|--------|------|------|
| NAT | Required (masquerade) | Not needed (NPT or native) |
| Address space | Scarce — NAT mandatory | Abundant — each WAN gets prefix |
| PCC | Standard mangle | Same mangle, IPv6 firewall tables |
| Default route | 0.0.0.0/0 | ::/0 |
| Gateway | ISP gateway IP | Link-local fe80:: or global |

---

## 12.2 IPv6 Address Plan

| Interface | IPv6 Prefix | Source |
|-----------|-------------|--------|
| ether1 (ISP-1) | 2001:db8:1::/64 | ISP-1 delegation |
| ether2 (ISP-2) | 2001:db8:2::/64 | ISP-2 delegation |
| ether3 (ISP-3) | 2001:db8:3::/64 | ISP-3 delegation |
| LAN | 2001:db8:100::/64 | ULA or ISP-1 delegation |

---

## 12.3 IPv6 PCC Configuration

```
/ipv6 firewall mangle
add chain=prerouting in-interface-list=LAN connection-mark=no-mark \
    dst-address-type=!local per-connection-classifier=both-addresses-and-ports:3/0 \
    action=mark-connection new-connection-mark=WAN1-conn passthrough=yes
add chain=prerouting in-interface-list=LAN connection-mark=no-mark \
    dst-address-type=!local per-connection-classifier=both-addresses-and-ports:3/1 \
    action=mark-connection new-connection-mark=WAN2-conn passthrough=yes
add chain=prerouting in-interface-list=LAN connection-mark=no-mark \
    dst-address-type=!local per-connection-classifier=both-addresses-and-ports:3/2 \
    action=mark-connection new-connection-mark=WAN3-conn passthrough=yes

add chain=prerouting connection-mark=WAN1-conn action=mark-routing new-routing-mark=to-WAN1 passthrough=yes
add chain=prerouting connection-mark=WAN2-conn action=mark-routing new-routing-mark=to-WAN2 passthrough=yes
add chain=prerouting connection-mark=WAN3-conn action=mark-routing new-routing-mark=to-WAN3 passthrough=yes
```

---

## 12.4 IPv6 Routes

```
/ipv6 route
add dst-address=::/0 gateway=2001:db8:1::1 routing-table=to-WAN1 distance=1
add dst-address=::/0 gateway=2001:db8:2::1 routing-table=to-WAN2 distance=1
add dst-address=::/0 gateway=2001:db8:3::1 routing-table=to-WAN3 distance=1
add dst-address=::/0 gateway=2001:db8:1::1 distance=1
add dst-address=::/0 gateway=2001:db8:2::1 distance=2
add dst-address=::/0 gateway=2001:db8:3::1 distance=3
```

---

## 12.5 NPT (Network Prefix Translation)

When LAN uses ULA (fd00::/48) and must present different ISP prefixes:

```
/ipv6 firewall nat
add chain=srcnat out-interface=ether1 action=src-nat to-address=2001:db8:1::/48
add chain=srcnat out-interface=ether2 action=src-nat to-address=2001:db8:2::/48
```

---

## 12.6 IPv6 Firewall

```
/ipv6 firewall filter
add chain=input connection-state=established,related action=accept
add chain=input connection-state=invalid action=drop
add chain=input protocol=icmpv6 action=accept
add chain=input in-interface-list=LAN action=accept
add chain=input in-interface-list=WAN action=drop
add chain=forward connection-state=established,related action=accept
add chain=forward in-interface-list=LAN out-interface-list=WAN action=accept
add chain=forward in-interface-list=WAN connection-state=new action=drop
```

---

## 12.7 DHCPv6 and SLAAC

```
/ipv6 nd
set [find interface=bridge-lan] advertise-dns=yes managed-address-configuration=no \
    other-configuration=yes

/ipv6 pool
add name=ipv6-pool prefix=2001:db8:100::/64 prefix-length=64

/ipv6 dhcp-server
add name=dhcpv6 interface=bridge-lan address-pool=ipv6-pool
```

---

**Next Chapter →** [Chapter 13: BGP Multi-Homing](../13-bgp-multihoming/README.md)
