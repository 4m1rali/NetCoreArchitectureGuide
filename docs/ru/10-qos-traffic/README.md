# Глава 10 — QoS & Traffic Prioritization

> Управление bandwidth и приоритизация трафика на нескольких WAN-каналах.

---

## 10.1 Зачем QoS важен в Multi-WAN

Без QoS одна тяжёлая загрузка на одном WAN может насытить канал и ухудшить VoIP, gaming и business-critical traffic — даже при идеальном PCC distribution.

| Проблема | QoS Solution |
|----------|--------------|
| Bulk download starves VoIP | Priority queue for UDP 5060/RTP |
| One user consumes all bandwidth | Per-user queue limits |
| WAN uplink saturated | PCQ rate limiting per subnet |
| Latency spikes during peak | Burst control + priority |

---

## 10.2 Типы очередей в MikroTik

| Тип очереди | Use Case | Multi-WAN |
|-------------|----------|-----------|
| **sfq** (Stochastic Fairness) | Default fair sharing | Per-interface |
| **pcq** (Per Connection) | Limit per user/IP | Best for ISP |
| **red** | Congestion avoidance | WAN interfaces |
| **fifo** | Simple bandwidth cap | Per-WAN total limit |
| **priority** | VoIP > Web > Bulk | LAN egress |

---

## 10.3 Per-WAN Bandwidth Limiting

```
/queue simple
add name=WAN1-limit target=ether1 max-limit=950M/950M comment="ISP-1 cap"
add name=WAN2-limit target=ether2 max-limit=480M/480M comment="ISP-2 cap"
add name=WAN3-limit target=ether3 max-limit=95M/95M comment="ISP-3 LTE cap"
```

Оставьте 5% headroom ниже advertised rate ISP, чтобы избежать drops на стороне провайдера.

---

## 10.4 Mangle-Based Traffic Classification

```
/ip firewall mangle
add chain=prerouting protocol=udp dst-port=5060,10000-20000 \
    action=mark-packet new-packet-mark=voip-pkt passthrough=yes
add chain=prerouting protocol=tcp dst-port=80,443 \
    action=mark-packet new-packet-mark=web-pkt passthrough=yes
add chain=prerouting protocol=tcp dst-port=21,22 \
    action=mark-packet new-packet-mark=bulk-pkt passthrough=yes
```

---

## 10.5 Priority Queue Tree

```
/queue tree
add name=WAN1-root parent=ether1 max-limit=950M
add name=WAN1-voip parent=WAN1-root packet-mark=voip-pkt priority=1 limit-at=50M
add name=WAN1-web parent=WAN1-root packet-mark=web-pkt priority=4 limit-at=200M
add name=WAN1-bulk parent=WAN1-root packet-mark=bulk-pkt priority=8 limit-at=100M
add name=WAN1-default parent=WAN1-root priority=6
```

Повторите для WAN2 и WAN3 с limits, скорректированными под ёмкость.

---

## 10.6 PCQ Per-User Fairness (ISP)

```
/queue type
add name=pcq-upload kind=pcq pcq-classifier=src-address pcq-rate=10M
add name=pcq-download kind=pcq pcq-classifier=dst-address pcq-rate=50M

/queue simple
add name=user-fair-upload target=ether1 queue=pcq-upload/pcq-upload max-limit=950M
add name=user-fair-download target=bridge-lan queue=pcq-download/pcq-download max-limit=950M
```

---

## 10.7 QoS + PCC Integration

| Правило | Причина |
|---------|---------|
| Mangle PCC rules BEFORE QoS marks | Connection classification first |
| QoS on out-interface (WAN) | Shape egress per ISP |
| Do not shape PCC marks | Shape by packet-mark instead |
| Keep VoIP on policy-routed WAN | QoS + policy routing together |

---

## 10.8 Burst and Burst Threshold

```
/queue simple
add name=burst-example target=ether1 max-limit=100M/100M \
    burst-limit=150M/150M burst-threshold=80M/80M burst-time=8s/8s
```

Позволяет короткие bursts выше sustained rate — полезно для web browsing patterns.

---

**Следующая глава →** [Глава 11: VPN over Multi-WAN](../11-vpn-multiwan/README.md)
