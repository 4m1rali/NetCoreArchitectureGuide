# Глава 6 — Debugging & Troubleshooting

> Анализ реальных сбоев и инструменты диагностики MikroTik.

---

## 6.1 Почему интернет постоянно отключается

### Корневые причины

| Причина | Симптом | Диагностика |
|---------|---------|-------------|
| Gateway flapping | Прерывистые outages 5–30s | `/ip route print detail` — маршруты toggling active/inactive |
| check-gateway false positive | Outages при congestion | Ping timeout на loaded link |
| DNS failure | «Connected», но страницы не загружаются | `/ping 8.8.8.8` работает, `/ping google.com` fails |
| Connection table full | Постепенная деградация | `/ip firewall connection print count-only` |
| ISP-side issue | Gateway reachable, нет internet | Ping gateway OK, ping 8.8.8.8 fails |
| NAT table exhaustion | Новые соединения не устанавливаются | `/ip firewall nat print` — high entry count |

### Диагностические команды

```
/ip route print detail where inactive
/ip route print detail where check-gateway
/ping 203.0.113.1 count=50
/ping 8.8.8.8 count=50
/tool netwatch print
/log print where topics~"route"
```

### Шаги устранения

1. Проверить статус маршрутов: `/ip route print detail`
2. Проверить доступность gateway: `/ping <gateway-ip> count=20`
3. Проверить доступность internet: `/ping 8.8.8.8 count=20`
4. Проверить route flapping в logs: `/log print`
5. Увеличить check-gateway timeout при false positives
6. Добавить Netwatch для upstream monitoring (8.8.8.8)

---

## 6.2 Почему Load Balancing становится нестабильным

### Корневые причины

| Причина | Симптом | Диагностика |
|---------|---------|-------------|
| Missing passthrough=yes | Random WAN per packet | `/ip firewall mangle print` — check passthrough |
| Mangle rule order wrong | Uneven distribution | `/ip firewall mangle print stats` |
| FastTrack bypassing mangle | Some connections unmarked | `/ip firewall filter print` — fasttrack rules |
| Connection mark not persisting | Sessions switch WAN mid-stream | `/ip firewall connection print` — check marks |
| One WAN route inactive | 50% traffic instead of 33% | `/ip route print` — inactive routes |
| Asymmetric routing | Downloads work, uploads fail | `/tool torch` — check both directions |

### Диагностические команды

```
/ip firewall mangle print stats
/ip firewall connection print where connection-mark!=""
/ip route print where routing-table~"to-WAN"
/tool torch interface=ether5 duration=30
/interface monitor-traffic ether1,ether2,ether3 once
```

### Проверка распределения

```
/ip firewall connection print count-only where connection-mark="WAN1-conn"
/ip firewall connection print count-only where connection-mark="WAN2-conn"
/ip firewall connection print count-only where connection-mark="WAN3-conn"
```

Ожидается: примерно равное количество по всем трём WAN marks.

### Шаги устранения

1. Проверить порядок mangle rules и passthrough setting
2. Отключить FastTrack если active (conflicts with PCC mangle)
3. Подтвердить, что все PCC routing table routes active
4. Проверить connection mark distribution (должно быть ~33/33/33)
5. Мониторить per-interface traffic для подтверждения balance

---

## 6.3 Почему возникает неверная маршрутизация

### Корневые причины

| Причина | Симптом | Диагностика |
|---------|---------|-------------|
| Missing routing table | Traffic uses main table | `/routing table print` |
| Routing mark not set | PCC ignored | `/ip firewall mangle print stats` |
| Recursive route missing | Gateway unreachable | `/ip route print detail where dst-address~"GW-IP"` |
| Scope/target-scope wrong | Route not used for forwarding | `/ip route print detail` — check scope |
| Distance conflict | Failover route overrides PCC | Compare main vs PCC table routes |
| VRF misconfiguration | Traffic in wrong FIB | `/routing table print` |

### Диагностические команды

```
/ip route print detail
/routing table print
/ip route print where routing-mark!=""
/ip firewall mangle print stats
/routing rule print
```

### Trace Route Decision

```
/tool sniffer quick interface=ether5 ip-protocol=tcp port=443
/tool traceroute 8.8.8.8
```

### Шаги устранения

1. Проверить наличие routing tables: `/routing table print`
2. Подтвердить, что PCC routes в correct tables (не main)
3. Проверить recursive gateway routes имеют scope=10
4. Проверить mangle routing-mark rules имеют traffic
5. Убедиться, что нет conflicting routes в main table с lower distance

---

## 6.4 Почему возникают конфликты NAT

### Корневые причины

| Причина | Симптом | Диагностика |
|---------|---------|-------------|
| Single masquerade for all WANs | Return traffic via wrong WAN | `/ip firewall nat print` |
| NAT before routing mark | Wrong out-interface selected | Rule order in nat chain |
| Duplicate NAT rules | Double translation | `/ip firewall nat print` — duplicate entries |
| Port collision | Random connection failures | Two WANs NAT same port simultaneously |
| Hairpin NAT missing | Internal access to public IP fails | No dstnat loopback rule |
| Connection mark mismatch | NAT applied but wrong WAN IP | Cross-reference conn mark and NAT interface |

### Диагностические команды

```
/ip firewall nat print
/ip firewall connection print where src-address~"192.168"
/tool sniffer quick interface=ether2 ip-address=198.51.100.2
/ip firewall nat print stats
```

### Шаги устранения

1. Обеспечить separate masquerade rules per out-interface
2. Проверить, что NAT out-interface matches PCC routing path
3. Проверить duplicate or conflicting NAT rules
4. Подтвердить connection tracking shows correct translated addresses
5. Добавить hairpin NAT если internal servers use public DNS

---

## 6.5 MikroTik Debug Toolkit

### Основные debug-команды

| Tool | Command | Purpose |
|------|---------|---------|
| Route status | `/ip route print detail` | Active/inactive routes, gateways |
| Connection table | `/ip firewall connection print` | Active sessions, marks, NAT |
| Mangle stats | `/ip firewall mangle print stats` | Rule hit counters |
| NAT stats | `/ip firewall nat print stats` | NAT rule hit counters |
| Traffic monitor | `/interface monitor-traffic <iface>` | Real-time bandwidth per WAN |
| Torch | `/tool torch <iface>` | Live connection analyzer |
| Sniffer | `/tool sniffer quick <iface>` | Packet capture |
| Traceroute | `/tool traceroute <ip>` | Path verification |
| Ping | `/ping <ip> count=50` | Latency and loss |
| Bandwidth test | `/tool bandwidth-test` | WAN throughput measurement |
| Netwatch | `/tool netwatch print` | Gateway monitor status |
| Logs | `/log print` | System events and errors |

### Debug Workflow

```
STEP 1: Verify physical layer
  /interface print stats
  /interface ethernet print

STEP 2: Verify IP and gateway
  /ip address print
  /ip route print detail

STEP 3: Verify PCC classification
  /ip firewall mangle print stats
  /ip firewall connection print count-only

STEP 4: Verify NAT
  /ip firewall nat print stats
  /ip firewall connection print where protocol=tcp

STEP 5: Verify traffic flow
  /interface monitor-traffic ether1,ether2,ether3
  /tool torch interface=ether5

STEP 6: Check logs
  /log print where topics~"firewall,route,system"
```

### Packet Sniffer для PCC Debug

```
/tool sniffer
set filter-interface=ether5 filter-ip-protocol=tcp \
    filter-connection-mark=WAN2-conn streaming-enabled=yes

/tool sniffer quick interface=ether5 duration=10
```

---

## 6.6 Quick Reference: проблема → решение

| Проблема | Quick Fix |
|----------|-----------|
| No internet at all | Check default route active, ping gateway |
| Internet via one WAN only | PCC mangle rules missing or disabled |
| Intermittent disconnects | check-gateway too aggressive, increase timeout |
| Slow browsing | DNS issue — check `/ip dns print` |
| VPN doesn't work | Policy route VPN to single WAN |
| VoIP choppy | Policy route VoIP to lowest-latency WAN |
| Upload works, download doesn't | NAT conflict — check per-WAN masquerade |
| High CPU | Too many mangle rules or connection table full |
| After reboot no balancing | Verify startup order, routing tables persist |
| One WAN always at 0% | Route inactive — check-gateway failing |

---

**Следующая глава →** [Глава 7: Engineering Analysis](../07-engineering-analysis/README.md)
