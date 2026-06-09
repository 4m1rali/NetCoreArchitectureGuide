# Глава 11 — VPN over Multi-WAN

> Развёртывание IPsec, WireGuard и L2TP на нескольких WAN-путях.

---

## 11.1 VPN и Multi-WAN — основное правило

**VPN tunnel — это одно соединение; его нельзя PCC-balanced между WAN.**

Каждый VPN tunnel должен быть привязан к **одному конкретному WAN interface**. Используйте policy routing для выбора WAN и разворачивайте redundant tunnels на разных WAN для resilience.

---

## 11.2 Архитектурный паттерн

```
                    ┌──────────────┐
                    │  VPN Server  │
                    │  (HQ/DMZ)    │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
        IPsec WAN1   WireGuard WAN2   L2TP WAN3
        (Primary)    (Backup)         (Mobile)
              │            │            │
              └────────────┼────────────┘
                           │
                    ┌──────▼───────┐
                    │   MikroTik   │
                    │  Branch CCR  │
                    └──────────────┘
```

---

## 11.3 IPsec через конкретный WAN

```
/ip ipsec peer
add name=peer-hq address=203.0.113.100 local-address=203.0.113.2 \
    exchange-mode=ike2

/ip ipsec identity
add peer=peer-hq auth-method=pre-shared-key secret=strong-preshared-key

/ip ipsec policy
add src-address=192.168.1.0/24 dst-address=10.0.0.0/24 peer=peer-hq \
    tunnel=yes action=encrypt

/ip firewall mangle
add chain=prerouting dst-address=203.0.113.100 action=mark-routing \
    new-routing-mark=to-WAN1 passthrough=yes comment="IPsec via ISP-1"
```

---

## 11.4 WireGuard через конкретный WAN

```
/interface wireguard
add name=wg-hq listen-port=51820 mtu=1420

/interface wireguard peers
add interface=wg-hq public-key="HQ_PUBLIC_KEY" endpoint-address=198.51.100.100 \
    endpoint-port=51820 allowed-address=10.0.0.0/24

/ip address
add address=10.0.0.2/24 interface=wg-hq

/ip firewall mangle
add chain=prerouting out-interface=wg-hq action=mark-routing \
    new-routing-mark=to-WAN2 passthrough=yes comment="WireGuard via ISP-2"
```

---

## 11.5 VPN Failover между WAN

Разверните два tunnel к одному HQ endpoint через разные WAN:

```
# Primary: IPsec via WAN1
# Backup: IPsec via WAN2 (disabled until WAN1 fails)

/tool netwatch
add host=203.0.113.1 interval=10s timeout=3s \
    down-script={
        /interface wireguard enable wg-hq-backup
        /ip route enable [find comment="VPN-backup-route"]
    } \
    up-script={
        /interface wireguard disable wg-hq-backup
        /ip route disable [find comment="VPN-backup-route"]
    }
```

---

## 11.6 Remote Access (Road Warrior) over Multi-WAN

Публикуйте VPN на всех публичных WAN IP:

```
# L2TP/IPsec on all WAN interfaces
/interface l2tp-server server
set enabled=yes max-mtu=1450 max-mru=1450

/ip firewall nat
add chain=dstnat in-interface=ether1 protocol=udp dst-port=500,4500,1701 action=accept
add chain=dstnat in-interface=ether2 protocol=udp dst-port=500,4500,1701 action=accept
```

Клиенты подключаются к любому reachable public IP.

---

## 11.7 VPN Performance Tuning

| Параметр | Значение | Причина |
|----------|----------|---------|
| MTU | 1400–1420 | Avoid fragmentation over PPPoE/LTE WAN |
| MSS clamp | 1360 | VPN header overhead |
| keepalive | 10s | Fast dead tunnel detection |
| FastTrack | Disabled on VPN traffic | Preserve mangle marks |

```
/ip firewall mangle
add chain=forward out-interface=wg-hq protocol=tcp tcp-flags=syn \
    action=change-mss new-mss=1360 passthrough=yes
```

---

## 11.8 Типичные ошибки VPN + Multi-WAN

| Ошибка | Последствие | Исправление |
|--------|-------------|-------------|
| VPN without policy route | Tunnel traffic PCC-balanced → breaks | Force routing-mark to specific WAN |
| NAT on IPsec | Double NAT breaks IKE | Accept IPsec before masquerade |
| Same tunnel on two WANs | IKE rekey conflicts | One tunnel per WAN |
| No MSS clamp | Intermittent large packet failure | Clamp MSS on VPN interface |

---

**Следующая глава →** [Глава 12: IPv6 Multi-WAN](../12-ipv6-multiwan/README.md)
