# Chapter 10 — QoS & Traffic Prioritization

> Bandwidth management and traffic prioritization across multiple WAN links.

---

## 10.1 Why QoS Matters in Multi-WAN

Without QoS, a single heavy download on one WAN can saturate the link and degrade VoIP, gaming, and business-critical traffic — even with perfect PCC distribution.

| Problem | QoS Solution |
|---------|-------------|
| Bulk download starves VoIP | Priority queue for UDP 5060/RTP |
| One user consumes all bandwidth | Per-user queue limits |
| WAN uplink saturated | PCQ rate limiting per subnet |
| Latency spikes during peak | Burst control + priority |

---

## 10.2 Queue Types in MikroTik

| Queue Type | Use Case | Multi-WAN |
|------------|----------|-----------|
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

Leave 5% headroom below ISP advertised rate to avoid ISP-side drops.

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

Repeat for WAN2 and WAN3 with capacity-adjusted limits.

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

| Rule | Reason |
|------|--------|
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

Allows short bursts above sustained rate — useful for web browsing patterns.

---

**Next Chapter →** [Chapter 11: VPN over Multi-WAN](../11-vpn-multiwan/README.md)
