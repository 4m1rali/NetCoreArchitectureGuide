# فصل ۱۲ — IPv6 Multi-WAN

> Dual-stack و IPv6-only Multi-WAN روی MikroTik RouterOS 7.

---

## ۱۲.۱ تفاوت‌های IPv6 Multi-WAN با IPv4

| جنبه | IPv4 | IPv6 |
|------|------|------|
| NAT | الزامی (masquerade) | لازم نیست (NPT یا Native) |
| فضای آدرس | کم — NAT اجباری | فراوان — هر WAN Prefix می‌گیرد |
| PCC | mangle استاندارد | همان mangle، جداول Firewall IPv6 |
| Default route | 0.0.0.0/0 | ::/0 |
| Gateway | IP Gateway ISP | Link-local fe80:: یا Global |

---

## ۱۲.۲ طرح آدرس IPv6

| Interface | Prefix IPv6 | منبع |
|-----------|-------------|------|
| ether1 (ISP-1) | 2001:db8:1::/64 | Delegation ISP-1 |
| ether2 (ISP-2) | 2001:db8:2::/64 | Delegation ISP-2 |
| ether3 (ISP-3) | 2001:db8:3::/64 | Delegation ISP-3 |
| LAN | 2001:db8:100::/64 | ULA یا Delegation ISP-1 |

---

## ۱۲.۳ پیکربندی PCC IPv6

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

## ۱۲.۴ Routeهای IPv6

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

## ۱۲.۵ NPT (Network Prefix Translation)

وقتی LAN از ULA (fd00::/48) استفاده می‌کند و باید Prefixهای ISP متفاوت ارائه دهد:

```
/ipv6 firewall nat
add chain=srcnat out-interface=ether1 action=src-nat to-address=2001:db8:1::/48
add chain=srcnat out-interface=ether2 action=src-nat to-address=2001:db8:2::/48
```

---

## ۱۲.۶ Firewall IPv6

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

## ۱۲.۷ DHCPv6 و SLAAC

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

**فصل بعد ←** [فصل ۱۳: BGP Multi-Homing](../13-bgp-multihoming/README.md)
