# Глава 5 — Полная конфигурация MikroTik

> Production-ready конфигурация RouterOS. Готова к copy-paste. Без inline-комментариев.

---

## 5.1 Настройка интерфейсов и IP

```
/interface list
add name=WAN
add name=LAN

/interface list member
add interface=ether1 list=WAN
add interface=ether2 list=WAN
add interface=ether3 list=WAN
add interface=ether5 list=LAN

/ip address
add address=203.0.113.2/30 interface=ether1 comment="ISP-1"
add address=198.51.100.2/30 interface=ether2 comment="ISP-2"
add address=192.0.2.2/30 interface=ether3 comment="ISP-3"
add address=192.168.1.1/24 interface=ether5 comment="LAN"
```

---

## 5.2 WAN Gateway и Recursive Routes

```
/ip route
add dst-address=203.0.113.1/32 gateway=ether1 scope=10 comment="ISP-1 GW route"
add dst-address=198.51.100.1/32 gateway=ether2 scope=10 comment="ISP-2 GW route"
add dst-address=192.0.2.1/32 gateway=ether3 scope=10 comment="ISP-3 GW route"
```

---

## 5.3 Таблицы маршрутизации

```
/routing table
add name=to-WAN1 fib
add name=to-WAN2 fib
add name=to-WAN3 fib
```

---

## 5.4 PCC Routing (Per-WAN Tables)

```
/ip route
add dst-address=0.0.0.0/0 gateway=203.0.113.1 routing-table=to-WAN1 distance=1 check-gateway=ping comment="PCC ISP-1"
add dst-address=0.0.0.0/0 gateway=198.51.100.1 routing-table=to-WAN2 distance=1 check-gateway=ping comment="PCC ISP-2"
add dst-address=0.0.0.0/0 gateway=192.0.2.1 routing-table=to-WAN3 distance=1 check-gateway=ping comment="PCC ISP-3"
```

---

## 5.5 Failover Routes (Main Table)

```
/ip route
add dst-address=0.0.0.0/0 gateway=203.0.113.1 distance=1 check-gateway=ping comment="Failover ISP-1"
add dst-address=0.0.0.0/0 gateway=198.51.100.1 distance=2 check-gateway=ping comment="Failover ISP-2"
add dst-address=0.0.0.0/0 gateway=192.0.2.1 distance=3 check-gateway=ping comment="Failover ISP-3"
```

---

## 5.6 ECMP Configuration (Alternative)

```
/ip route
add dst-address=0.0.0.0/0 gateway=203.0.113.1,198.51.100.1,192.0.2.1 distance=1 check-gateway=ping comment="ECMP all WANs"

/routing settings
set ecmp-per-connection=yes
```

---

## 5.7 PCC Mangle Rules

```
/ip firewall mangle
add chain=prerouting in-interface-list=LAN connection-mark=no-mark dst-address-type=!local per-connection-classifier=both-addresses-and-ports:3/0 action=mark-connection new-connection-mark=WAN1-conn passthrough=yes comment="PCC WAN1"
add chain=prerouting in-interface-list=LAN connection-mark=no-mark dst-address-type=!local per-connection-classifier=both-addresses-and-ports:3/1 action=mark-connection new-connection-mark=WAN2-conn passthrough=yes comment="PCC WAN2"
add chain=prerouting in-interface-list=LAN connection-mark=no-mark dst-address-type=!local per-connection-classifier=both-addresses-and-ports:3/2 action=mark-connection new-connection-mark=WAN3-conn passthrough=yes comment="PCC WAN3"
add chain=prerouting in-interface-list=LAN connection-mark=WAN1-conn action=mark-routing new-routing-mark=to-WAN1 passthrough=yes comment="Route mark WAN1"
add chain=prerouting in-interface-list=LAN connection-mark=WAN2-conn action=mark-routing new-routing-mark=to-WAN2 passthrough=yes comment="Route mark WAN2"
add chain=prerouting in-interface-list=LAN connection-mark=WAN3-conn action=mark-routing new-routing-mark=to-WAN3 passthrough=yes comment="Route mark WAN3"
```

---

## 5.8 NAT Configuration

```
/ip firewall nat
add chain=srcnat out-interface=ether1 action=masquerade comment="NAT ISP-1"
add chain=srcnat out-interface=ether2 action=masquerade comment="NAT ISP-2"
add chain=srcnat out-interface=ether3 action=masquerade comment="NAT ISP-3"
```

### Port Forwarding (Per-WAN Example)

```
/ip firewall nat
add chain=dstnat in-interface=ether1 protocol=tcp dst-port=443 action=dst-nat to-addresses=192.168.1.10 to-ports=443 comment="HTTPS via ISP-1"
add chain=dstnat in-interface=ether2 protocol=tcp dst-port=443 action=dst-nat to-addresses=192.168.1.10 to-ports=443 comment="HTTPS via ISP-2"
```

---

## 5.9 Firewall — Basic Security

```
/ip firewall filter
add chain=input connection-state=established,related action=accept comment="Accept established input"
add chain=input connection-state=invalid action=drop comment="Drop invalid input"
add chain=input protocol=icmp action=accept comment="Accept ICMP input"
add chain=input in-interface-list=LAN action=accept comment="Accept LAN management"
add chain=input in-interface-list=WAN action=drop comment="Drop WAN input"

add chain=forward connection-state=established,related action=accept comment="Accept established forward"
add chain=forward connection-state=invalid action=drop comment="Drop invalid forward"
add chain=forward in-interface-list=LAN out-interface-list=WAN action=accept comment="LAN to WAN"
add chain=forward in-interface-list=WAN out-interface-list=LAN connection-state=new action=drop comment="Block new WAN to LAN"
add chain=forward in-interface-list=WAN out-interface-list=WAN action=drop comment="Anti-loop WAN to WAN"
```

---

## 5.10 Failover — Netwatch Scripts

```
/tool netwatch
add host=203.0.113.1 interval=10s timeout=3s \
    up-script={/ip route enable [find comment="PCC ISP-1"]} \
    down-script={/ip route disable [find comment="PCC ISP-1"]} \
    comment="Monitor ISP-1"

add host=198.51.100.1 interval=10s timeout=3s \
    up-script={/ip route enable [find comment="PCC ISP-2"]} \
    down-script={/ip route disable [find comment="PCC ISP-2"]} \
    comment="Monitor ISP-2"

add host=192.0.2.1 interval=10s timeout=3s \
    up-script={/ip route enable [find comment="PCC ISP-3"]} \
    down-script={/ip route disable [find comment="PCC ISP-3"]} \
    comment="Monitor ISP-3"
```

---

## 5.11 DNS Configuration

```
/ip dns
set allow-remote-requests=yes servers=8.8.8.8,1.1.1.1

/ip dhcp-server network
set [find] dns-server=192.168.1.1
```

---

## 5.12 DHCP Server (LAN)

```
/ip pool
add name=dhcp-pool ranges=192.168.1.100-192.168.1.254

/ip dhcp-server
add name=dhcp1 interface=ether5 address-pool=dhcp-pool lease-time=1d

/ip dhcp-server network
add address=192.168.1.0/24 gateway=192.168.1.1 dns-server=192.168.1.1
```

---

## 5.13 System Hardening

```
/ip service
set telnet disabled=yes
set ftp disabled=yes
set www disabled=yes
set api disabled=yes
set api-ssl disabled=yes

/ip settings
set rp-filter=strict

/system ntp client
set enabled=yes
add address=pool.ntp.org
```

---

## 5.14 PPPoE WAN Configuration

```
/interface pppoe-client
add name=pppoe-isp1 interface=ether1 user=customer@isp1 password=secret \
    add-default-route=no use-peer-dns=no disabled=no
add name=pppoe-isp2 interface=ether2 user=customer@isp2 password=secret \
    add-default-route=no use-peer-dns=no disabled=no

/interface list member
add interface=pppoe-isp1 list=WAN
add interface=pppoe-isp2 list=WAN

/ip route
add dst-address=0.0.0.0/0 gateway=pppoe-isp1 routing-table=to-WAN1 distance=1 check-gateway=ping
add dst-address=0.0.0.0/0 gateway=pppoe-isp2 routing-table=to-WAN2 distance=1 check-gateway=ping

/ip firewall mangle
add chain=forward protocol=tcp tcp-flags=syn out-interface-list=WAN \
    action=change-mss new-mss=1440 passthrough=yes comment="MSS clamp PPPoE"

/ip firewall nat
add chain=srcnat out-interface=pppoe-isp1 action=masquerade
add chain=srcnat out-interface=pppoe-isp2 action=masquerade
```

---

## 5.15 Policy Routing (VoIP + PCC)

```
/ip firewall mangle
add chain=prerouting src-address=192.168.1.240/28 protocol=udp dst-port=5060,10000-20000 \
    action=mark-connection new-connection-mark=WAN1-conn passthrough=yes comment="VoIP policy"
add chain=prerouting src-address=192.168.1.0/28 protocol=tcp dst-port=22,8291 \
    action=mark-connection new-connection-mark=WAN3-conn passthrough=yes comment="Mgmt policy"
```

Разместите эти правила **перед** PCC classifier rules в mangle chain.

---

## 5.16 QoS Basic Setup

```
/ip firewall mangle
add chain=prerouting protocol=udp dst-port=5060,10000-20000 \
    action=mark-packet new-packet-mark=voip-pkt passthrough=yes
add chain=prerouting protocol=tcp dst-port=80,443 \
    action=mark-packet new-packet-mark=web-pkt passthrough=yes

/queue tree
add name=WAN1-root parent=ether1 max-limit=950M
add name=WAN1-voip parent=WAN1-root packet-mark=voip-pkt priority=1 limit-at=50M
add name=WAN1-web parent=WAN1-root packet-mark=web-pkt priority=4
add name=WAN1-other parent=WAN1-root priority=6
```

---

## 5.17 Hairpin NAT

```
/ip firewall nat
add chain=dstnat in-interface-list=LAN dst-address=203.0.113.2 protocol=tcp \
    dst-port=80,443 action=dst-nat to-addresses=192.168.1.10
add chain=dstnat in-interface-list=LAN dst-address=198.51.100.2 protocol=tcp \
    dst-port=80,443 action=dst-nat to-addresses=192.168.1.10
add chain=srcnat in-interface-list=LAN dst-address=192.168.1.10 action=masquerade
```

---

## 5.18 Weighted PCC (70/30 Two-WAN)

```
/ip firewall mangle
add chain=prerouting in-interface-list=LAN connection-mark=no-mark dst-address-type=!local per-connection-classifier=both-addresses-and-ports:10/0 action=mark-connection new-connection-mark=WAN1-conn passthrough=yes
add chain=prerouting in-interface-list=LAN connection-mark=no-mark dst-address-type=!local per-connection-classifier=both-addresses-and-ports:10/1 action=mark-connection new-connection-mark=WAN1-conn passthrough=yes
add chain=prerouting in-interface-list=LAN connection-mark=no-mark dst-address-type=!local per-connection-classifier=both-addresses-and-ports:10/2 action=mark-connection new-connection-mark=WAN1-conn passthrough=yes
add chain=prerouting in-interface-list=LAN connection-mark=no-mark dst-address-type=!local per-connection-classifier=both-addresses-and-ports:10/3 action=mark-connection new-connection-mark=WAN1-conn passthrough=yes
add chain=prerouting in-interface-list=LAN connection-mark=no-mark dst-address-type=!local per-connection-classifier=both-addresses-and-ports:10/4 action=mark-connection new-connection-mark=WAN1-conn passthrough=yes
add chain=prerouting in-interface-list=LAN connection-mark=no-mark dst-address-type=!local per-connection-classifier=both-addresses-and-ports:10/5 action=mark-connection new-connection-mark=WAN1-conn passthrough=yes
add chain=prerouting in-interface-list=LAN connection-mark=no-mark dst-address-type=!local per-connection-classifier=both-addresses-and-ports:10/6 action=mark-connection new-connection-mark=WAN1-conn passthrough=yes
add chain=prerouting in-interface-list=LAN connection-mark=no-mark dst-address-type=!local per-connection-classifier=both-addresses-and-ports:10/7 action=mark-connection new-connection-mark=WAN2-conn passthrough=yes
add chain=prerouting in-interface-list=LAN connection-mark=no-mark dst-address-type=!local per-connection-classifier=both-addresses-and-ports:10/8 action=mark-connection new-connection-mark=WAN2-conn passthrough=yes
add chain=prerouting in-interface-list=LAN connection-mark=no-mark dst-address-type=!local per-connection-classifier=both-addresses-and-ports:10/9 action=mark-connection new-connection-mark=WAN2-conn passthrough=yes
```

---

## 5.19 Full Export Template (Complete Script)

```
# === INTERFACE SETUP ===
/interface list
add name=WAN
add name=LAN
/interface list member
add interface=ether1 list=WAN
add interface=ether2 list=WAN
add interface=ether3 list=WAN
add interface=ether5 list=LAN

# === IP ADDRESSES ===
/ip address
add address=203.0.113.2/30 interface=ether1
add address=198.51.100.2/30 interface=ether2
add address=192.0.2.2/30 interface=ether3
add address=192.168.1.1/24 interface=ether5

# === ROUTING TABLES ===
/routing table
add name=to-WAN1 fib
add name=to-WAN2 fib
add name=to-WAN3 fib

# === GATEWAY ROUTES ===
/ip route
add dst-address=203.0.113.1/32 gateway=ether1 scope=10
add dst-address=198.51.100.1/32 gateway=ether2 scope=10
add dst-address=192.0.2.1/32 gateway=ether3 scope=10

# === PCC ROUTES ===
add dst-address=0.0.0.0/0 gateway=203.0.113.1 routing-table=to-WAN1 distance=1 check-gateway=ping comment="PCC ISP-1"
add dst-address=0.0.0.0/0 gateway=198.51.100.1 routing-table=to-WAN2 distance=1 check-gateway=ping comment="PCC ISP-2"
add dst-address=0.0.0.0/0 gateway=192.0.2.1 routing-table=to-WAN3 distance=1 check-gateway=ping comment="PCC ISP-3"

# === FAILOVER ROUTES ===
add dst-address=0.0.0.0/0 gateway=203.0.113.1 distance=1 check-gateway=ping comment="Failover ISP-1"
add dst-address=0.0.0.0/0 gateway=198.51.100.1 distance=2 check-gateway=ping comment="Failover ISP-2"
add dst-address=0.0.0.0/0 gateway=192.0.2.1 distance=3 check-gateway=ping comment="Failover ISP-3"

# === PCC MANGLE ===
/ip firewall mangle
add chain=prerouting in-interface-list=LAN connection-mark=no-mark dst-address-type=!local per-connection-classifier=both-addresses-and-ports:3/0 action=mark-connection new-connection-mark=WAN1-conn passthrough=yes
add chain=prerouting in-interface-list=LAN connection-mark=no-mark dst-address-type=!local per-connection-classifier=both-addresses-and-ports:3/1 action=mark-connection new-connection-mark=WAN2-conn passthrough=yes
add chain=prerouting in-interface-list=LAN connection-mark=no-mark dst-address-type=!local per-connection-classifier=both-addresses-and-ports:3/2 action=mark-connection new-connection-mark=WAN3-conn passthrough=yes
add chain=prerouting connection-mark=WAN1-conn action=mark-routing new-routing-mark=to-WAN1 passthrough=yes
add chain=prerouting connection-mark=WAN2-conn action=mark-routing new-routing-mark=to-WAN2 passthrough=yes
add chain=prerouting connection-mark=WAN3-conn action=mark-routing new-routing-mark=to-WAN3 passthrough=yes

# === NAT ===
/ip firewall nat
add chain=srcnat out-interface=ether1 action=masquerade
add chain=srcnat out-interface=ether2 action=masquerade
add chain=srcnat out-interface=ether3 action=masquerade

# === FIREWALL ===
/ip firewall filter
add chain=input connection-state=established,related action=accept
add chain=input connection-state=invalid action=drop
add chain=input protocol=icmp action=accept
add chain=input in-interface-list=LAN action=accept
add chain=input in-interface-list=WAN action=drop
add chain=forward connection-state=established,related action=accept
add chain=forward connection-state=invalid action=drop
add chain=forward in-interface-list=LAN out-interface-list=WAN action=accept
add chain=forward in-interface-list=WAN out-interface-list=LAN connection-state=new action=drop
add chain=forward in-interface-list=WAN out-interface-list=WAN action=drop

# === NETWATCH ===
/tool netwatch
add host=203.0.113.1 interval=10s timeout=3s down-script={/ip route disable [find comment="PCC ISP-1"]} up-script={/ip route enable [find comment="PCC ISP-1"]}
add host=198.51.100.1 interval=10s timeout=3s down-script={/ip route disable [find comment="PCC ISP-2"]} up-script={/ip route enable [find comment="PCC ISP-2"]}
add host=192.0.2.1 interval=10s timeout=3s down-script={/ip route disable [find comment="PCC ISP-3"]} up-script={/ip route enable [find comment="PCC ISP-3"]}
```

---

## 5.20 Порядок конфигурации (Deployment Sequence)

| Шаг | Раздел | Действие |
|-----|--------|----------|
| 1 | 5.1 | Interface lists и IP addresses |
| 2 | 5.2 | Recursive gateway routes |
| 3 | 5.3 | Создание routing tables |
| 4 | 5.4 | PCC per-WAN routes |
| 5 | 5.5 | Failover main table routes |
| 6 | 5.7 | PCC mangle rules |
| 7 | 5.15 | Policy routing rules (if needed) |
| 8 | 5.8 | NAT rules |
| 9 | 5.9 | Firewall rules |
| 10 | 5.10 | Netwatch monitoring |
| 11 | 5.16 | QoS (if needed) |
| 12 | 5.11–5.13 | DNS, DHCP, hardening |
| 13 | 5.19 | Full export verification |

---

**Следующая глава →** [Глава 6: Debugging & Troubleshooting](../06-debugging-troubleshooting/README.md)
