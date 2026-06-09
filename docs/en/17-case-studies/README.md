# Chapter 17 — Real-World Case Studies

> Production deployment scenarios with architecture decisions and lessons learned.

---

## Case Study 1 — ISP WISP (300 Subscribers)

### Environment

| Item | Detail |
|------|--------|
| Router | MikroTik CCR2116-12G-4S+ |
| WAN-1 | Fiber 1Gbps (primary upstream) |
| WAN-2 | Wireless backhaul 500Mbps (secondary) |
| WAN-3 | LTE 50Mbps (emergency backup) |
| Subscribers | 300 PPPoE customers |
| Challenge | Even traffic distribution without session drops |

### Solution

- PCC 2-way (WAN1 + WAN2) — LTE excluded from PCC
- Per-WAN masquerade for PPPoE NAT
- PCQ per-subscriber rate limiting
- Netwatch on both primary gateways
- LTE failover distance=3 only

### Results

| Metric | Before | After |
|--------|--------|-------|
| Uptime | 97.2% | 99.7% |
| Avg latency | 45ms | 28ms |
| Complaints/month | 40 | 5 |
| WAN utilization balance | 90/10 | 52/48 |

### Lesson Learned

> LTE must never be in PCC classifier — one Netflix stream consumed entire monthly LTE quota in 4 hours.

---

## Case Study 2 — Enterprise HQ (200 Employees)

### Environment

| Item | Detail |
|------|--------|
| Router | MikroTik RB4011iGS+RM |
| WAN-1 | MPLS 200Mbps (corporate) |
| WAN-2 | Broadband 1Gbps (internet) |
| WAN-3 | 4G LTE 100Mbps (backup) |
| Services | VoIP, ERP, Microsoft 365, guest WiFi |
| Challenge | VoIP quality + guest isolation |

### Solution

- Policy routing: VoIP subnet → WAN1 (MPLS)
- Policy routing: ERP → WAN1
- PCC 2-way for general traffic (WAN2 + WAN3)
- Separate VLAN for guest WiFi with PCC on WAN2 only
- QoS priority queue for VoIP on WAN1

### Results

| Metric | Value |
|--------|-------|
| VoIP MOS score | 4.2 (excellent) |
| ERP latency | 12ms (MPLS) |
| Guest bandwidth | Capped 200Mbps |
| Failover time | 8 seconds (LTE) |

### Lesson Learned

> Policy routing rules must be placed **above** PCC rules. Initial deployment had VoIP going through PCC — calls dropped every 30 seconds.

---

## Case Study 3 — Datacenter Edge (BGP)

### Environment

| Item | Detail |
|------|--------|
| Router | MikroTik CCR2216-1G-12XS-2XQ |
| ASN | 65050 (private) with PI space 203.0.113.0/24 |
| WAN-1 | ISP-A BGP peer (10G) |
| WAN-2 | ISP-B BGP peer (10G) |
| Challenge | Inbound + outbound path control |

### Solution

- BGP to both ISPs advertising 203.0.113.0/24
- AS-PATH prepend on ISP-B (backup preference)
- ECMP for outbound (no NAT — public IPs)
- BGP route filtering — accept default only
- No PCC needed

### Results

| Metric | Value |
|--------|-------|
| Inbound control | Full (BGP path selection) |
| Outbound ECMP | 50/50 across 10G links |
| Failover | < 30 seconds (BGP hold timer) |
| Single stream max | 10Gbps (one path) |

---

## Case Study 4 — Branch Office (25 Users)

### Environment

| Item | Detail |
|------|--------|
| Router | MikroTik hEX S (RB760iGS) |
| WAN-1 | Cable 200Mbps |
| WAN-2 | DSL 50Mbps |
| Challenge | Limited router CPU for mangle |

### Solution

- Simple failover only (no PCC) — cable primary, DSL backup
- Weighted: cable handles 100% until failure
- Basic masquerade on active WAN
- check-gateway=ping on both routes

### Results

| Metric | Value |
|--------|-------|
| CPU usage | 15% (no mangle overhead) |
| Failover time | 12 seconds |
| Monthly cost savings | $0 (DSL already contracted) |

### Lesson Learned

> Not every deployment needs PCC. For < 50 users with asymmetric WAN speeds, failover-only is simpler and more stable on low-end hardware.

---

## Case Study 5 — Hotel (150 Rooms + Guest WiFi)

### Environment

| Item | Detail |
|------|--------|
| Router | MikroTik CCR2004-16G-2S+ |
| WAN-1 | Fiber 500Mbps |
| WAN-2 | Fiber 500Mbps (different ISP) |
| WAN-3 | LTE 100Mbps backup |
| Challenge | 150 concurrent streaming sessions |

### Solution

- PCC 3-way equal distribution
- Per-room PCQ rate limit (20Mbps down / 5Mbps up)
- DNS forced through router cache
- Hairpin NAT for internal booking system
- Hotspot on guest VLAN with PCC

### Results

| Metric | Value |
|--------|-------|
| Concurrent streams supported | 120+ |
| PCC balance | 34/33/33 |
| Guest complaints | Near zero |
| Failover during ISP maintenance | Transparent (8s) |

---

## Decision Framework from Case Studies

| Users | WAN Types | Recommended |
|-------|-----------|-------------|
| < 50 | Any | Failover only |
| 50–200 | 2–3 similar speed | PCC + Failover |
| 200–1000 | 2–3 + LTE backup | PCC + Policy + Failover |
| 1000+ | BGP capable | BGP or PCC on CCR |
| ISP/WISP | Multiple upstreams | PCC + PCQ + Netwatch |

---

**Next →** [Appendix A: Glossary](../appendix/glossary.md)
