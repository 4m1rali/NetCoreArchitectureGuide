# Приложение B — Quick Reference Cheat Sheet

---

## PCC 3-WAN (Copy-Paste)

```
/routing table add name=to-WAN1 fib
/routing table add name=to-WAN2 fib
/routing table add name=to-WAN3 fib

/ip route
add dst-address=0.0.0.0/0 gateway=GW1 routing-table=to-WAN1 distance=1 check-gateway=ping
add dst-address=0.0.0.0/0 gateway=GW2 routing-table=to-WAN2 distance=1 check-gateway=ping
add dst-address=0.0.0.0/0 gateway=GW3 routing-table=to-WAN3 distance=1 check-gateway=ping
add dst-address=0.0.0.0/0 gateway=GW1 distance=1 check-gateway=ping
add dst-address=0.0.0.0/0 gateway=GW2 distance=2 check-gateway=ping
add dst-address=0.0.0.0/0 gateway=GW3 distance=3 check-gateway=ping

/ip firewall mangle
add chain=prerouting in-interface-list=LAN connection-mark=no-mark dst-address-type=!local per-connection-classifier=both-addresses-and-ports:3/0 action=mark-connection new-connection-mark=WAN1-conn passthrough=yes
add chain=prerouting in-interface-list=LAN connection-mark=no-mark dst-address-type=!local per-connection-classifier=both-addresses-and-ports:3/1 action=mark-connection new-connection-mark=WAN2-conn passthrough=yes
add chain=prerouting in-interface-list=LAN connection-mark=no-mark dst-address-type=!local per-connection-classifier=both-addresses-and-ports:3/2 action=mark-connection new-connection-mark=WAN3-conn passthrough=yes
add chain=prerouting connection-mark=WAN1-conn action=mark-routing new-routing-mark=to-WAN1 passthrough=yes
add chain=prerouting connection-mark=WAN2-conn action=mark-routing new-routing-mark=to-WAN2 passthrough=yes
add chain=prerouting connection-mark=WAN3-conn action=mark-routing new-routing-mark=to-WAN3 passthrough=yes

/ip firewall nat
add chain=srcnat out-interface=WAN1 action=masquerade
add chain=srcnat out-interface=WAN2 action=masquerade
add chain=srcnat out-interface=WAN3 action=masquerade
```

---

## Debug One-Liners

| Задача | Команда |
|--------|---------|
| Статус маршрутов | `/ip route print detail` |
| Активные соединения | `/ip firewall connection print count-only` |
| Распределение PCC | `/ip firewall connection print count-only where connection-mark="WAN1-conn"` |
| Счётчики mangle | `/ip firewall mangle print stats` |
| Трафик per-WAN | `/interface monitor-traffic ether1,ether2,ether3 once` |
| Проверка шлюза | `/ping GW1 count=20` |
| Проверка интернета | `/ping 8.8.8.8 interface=ether1 count=20` |
| Live connections | `/tool torch interface=bridge-lan` |
| Backup конфигурации | `/export file=backup` |

---

## Выбор метода

| Потребность | Использовать |
|-------------|--------------|
| NAT + balance | PCC |
| No NAT + balance | ECMP |
| Backup path | Failover |
| VoIP on best WAN | Policy Routing |
| Own IP + ASN | BGP |
| < 50 users, simple | Failover only |

---

## Типичные исправления

| Проблема | Исправление |
|----------|-------------|
| Sessions break | Disable FastTrack, use PCC not ECMP |
| One WAN 0% | Check route inactive, check-gateway |
| NAT wrong IP | Per-interface masquerade |
| VoIP drops | Policy route above PCC |
| Large downloads fail | MSS clamp + check MTU |
| High CPU | Reduce mangle rules, upgrade hardware |

---

## RouterOS 6 vs 7 Differences

| Функция | ROS 6 | ROS 7 |
|---------|-------|-------|
| Routing tables | `/routing mark` + `mangle` | `/routing table` + `routing-table=` |
| BGP | `/routing bgp instance` | `/routing bgp template` + `connection` |
| Firewall | `/ip firewall` | `/ip firewall` + `/ipv6 firewall` |
| ECMP per-conn | Not available | `ecmp-per-connection=yes` |
| VRF | Limited | Full VRF support |

---

**[← Назад к индексу](../../README.md)**
