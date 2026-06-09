# Chapter 4 — Production Design

> Real-world 3-ISP Multi-WAN architecture with complete traffic flow analysis.

---

## 4.1 Scenario Definition

### Infrastructure

| Component | Detail |
|-----------|--------|
| Router | MikroTik CCR2004-16G-2S+ (or RB4011) |
| RouterOS | 7.x |
| ISP-1 | Fiber 1Gbps — Primary (low latency) |
| ISP-2 | Metro Ethernet 500Mbps — Secondary |
| ISP-3 | LTE Backup 100Mbps — Tertiary |
| LAN | 192.168.1.0/24 — 200 users + servers |
| Services | Web server, VoIP, file server, 200 workstations |

### IP Address Plan

| Interface | IP Address | Network |
|-----------|-----------|---------|
| ether1 (WAN1/ISP-1) | 203.0.113.2/30 | 203.0.113.0/30 |
| ether2 (WAN2/ISP-2) | 198.51.100.2/30 | 198.51.100.0/30 |
| ether3 (WAN3/ISP-3) | 192.0.2.2/30 | 192.0.2.0/30 |
| ether5 (LAN) | 192.168.1.1/24 | 192.168.1.0/24 |

| Gateway | IP Address |
|---------|-----------|
| ISP-1 GW | 203.0.113.1 |
| ISP-2 GW | 198.51.100.1 |
| ISP-3 GW | 192.0.2.1 |

---

## 4.2 Network Topology

```
                          INTERNET
                             │
          ┌──────────────────┼──────────────────┐
          │                  │                  │
    ┌─────┴─────┐     ┌───────┴──────┐    ┌─────┴─────┐
    │   ISP-1   │     │    ISP-2     │    │   ISP-3   │
    │  Fiber 1G │     │  Metro 500M  │    │  LTE 100M │
    │203.0.113.1│     │198.51.100.1  │    │ 192.0.2.1 │
    └─────┬─────┘     └───────┬──────┘    └─────┬─────┘
          │ ether1            │ ether2          │ ether3
          │                   │                 │
    ┌─────┴───────────────────┴─────────────────┴─────┐
    │                                                    │
    │              MikroTik CCR2004                      │
    │              RouterOS 7.x                          │
    │                                                    │
    │   ┌─────────┐  ┌──────────┐  ┌──────────────┐    │
    │   │ Mangle  │  │ Routing  │  │   Firewall   │    │
    │   │  PCC    │  │  Tables  │  │  NAT + Filter│    │
    │   └─────────┘  └──────────┘  └──────────────┘    │
    │                                                    │
    │                    ether5                          │
    └────────────────────┬───────────────────────────────┘
                         │
              ┌──────────┴──────────┐
              │    LAN Switch       │
              │  192.168.1.0/24    │
              └──────────┬──────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
   ┌────┴────┐    ┌──────┴──────┐   ┌────┴────┐
   │  Users  │    │ Web Server  │   │  VoIP   │
   │ .10-.200│    │  .10:80/443 │   │  .250   │
   └─────────┘    └─────────────┘   └─────────┘
```

---

## 4.3 Traffic Flow Diagram

### Outbound Traffic Flow (PCC)

```
[Client 192.168.1.50]
        │
        ▼
[ether5 LAN] ──→ [Bridge/LAN interface-list]
        │
        ▼
[Firewall FILTER: forward] ── accept from LAN
        │
        ▼
[Mangle PREROUTING]
   ├── PCC classifier → connection-mark: WAN2-conn
   └── routing-mark: to-WAN2
        │
        ▼
[Connection Tracking] ── CREATE entry
        │
        ▼
[Routing Table: to-WAN2]
   └── 0.0.0.0/0 via 198.51.100.1 → ACTIVE
        │
        ▼
[NAT SRCNAT] ── masquerade out-interface=ether2
   └── 192.168.1.50 → 198.51.100.2
        │
        ▼
[ether2 WAN2] ──→ [ISP-2] ──→ [Internet]
```

### Inbound Traffic Flow (Return)

```
[Internet] ──→ [ISP-2] ──→ [ether2]
        │
        ▼
[Connection Tracking] ── LOOKUP → ESTABLISHED
   └── connection-mark: WAN2-conn
        │
        ▼
[NAT DSTNAT reverse] ── 198.51.100.2 → 192.168.1.50
        │
        ▼
[Routing] ── 192.168.1.0/24 via ether5 (connected)
        │
        ▼
[ether5 LAN] ──→ [Client 192.168.1.50]
```

---

## 4.4 Routing Decision Flow

```
                    ┌─────────────────┐
                    │  Packet Arrives  │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ Connection-mark  │
                    │    exists?       │
                    └────┬──────┬─────┘
                         │      │
                    YES  │      │  NO
                         │      │
              ┌──────────▼┐  ┌──▼──────────────┐
              │ Use stored│  │ PCC Classifier   │
              │ mark from │  │ hash → assign    │
              │ conn table│  │ connection-mark  │
              └──────────┬┘  └──┬──────────────┘
                         │      │
                    ┌────▼──────▼─────┐
                    │  routing-mark    │
                    │  set from conn   │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
        ┌─────▼─────┐  ┌────▼────┐  ┌──────▼──────┐
        │ to-WAN1  │  │ to-WAN2 │  │  to-WAN3   │
        │ table    │  │ table   │  │  table     │
        └─────┬─────┘  └────┬────┘  └──────┬──────┘
              │              │              │
        ┌─────▼─────┐  ┌────▼────┐  ┌──────▼──────┐
        │ GW .113.1 │  │GW .100.1│  │  GW .2.1   │
        │ check-gw  │  │check-gw │  │  check-gw  │
        │ ACTIVE?   │  │ACTIVE?  │  │  ACTIVE?   │
        └─────┬─────┘  └────┬────┘  └──────┬──────┘
              │              │              │
         YES  │         YES  │         YES  │
              ▼              ▼              ▼
         [ether1]       [ether2]       [ether3]
              │              │              │
         NO → DROP    NO → DROP     NO → DROP
```

---

## 4.5 NAT Flow Explanation

### Per-WAN NAT Architecture

```
Connection marked WAN1-conn:
  → Route via to-WAN1 → egress ether1
  → NAT: action=masquerade out-interface=ether1
  → Source: 192.168.1.x → 203.0.113.2

Connection marked WAN2-conn:
  → Route via to-WAN2 → egress ether2
  → NAT: action=masquerade out-interface=ether2
  → Source: 192.168.1.x → 198.51.100.2

Connection marked WAN3-conn:
  → Route via to-WAN3 → egress ether3
  → NAT: action=masquerade out-interface=ether3
  → Source: 192.168.1.x → 192.0.2.2
```

### NAT + PCC Binding

| Step | Action | Result |
|------|--------|--------|
| 1 | PCC assigns WAN2-conn | Connection bound to ISP-2 |
| 2 | Routing uses to-WAN2 | Packet exits ether2 |
| 3 | NAT masquerade on ether2 | Public IP = 198.51.100.2 |
| 4 | Return packet to 198.51.100.2 | Connection table match |
| 5 | NAT reverse | Destination = 192.168.1.x |
| 6 | Route to LAN | Delivered to client |

---

## 4.6 Packet Journey — Complete Example

**Request:** User `192.168.1.50` opens `https://google.com`

| Step | Location | Action | Detail |
|------|----------|--------|--------|
| 1 | Client | TCP SYN sent | src=192.168.1.50:49821 dst=142.250.80.46:443 |
| 2 | ether5 | Frame received | Destination MAC = router LAN MAC |
| 3 | Filter forward | Rule match | `in-interface-list=LAN action=accept` |
| 4 | Mangle prerouting | PCC classify | hash(50+142.250.80.46+49821+443) mod 3 = 1 → WAN2-conn |
| 5 | Mangle prerouting | Routing mark | connection-mark=WAN2-conn → routing-mark=to-WAN2 |
| 6 | Conntrack | New entry | state=new, mark=WAN2-conn |
| 7 | Routing | Table lookup | to-WAN2: 0.0.0.0/0 via 198.51.100.1 ✓ ACTIVE |
| 8 | NAT srcnat | Masquerade | 192.168.1.50:49821 → 198.51.100.2:49821 |
| 9 | ether2 | Transmit | To ISP-2 gateway |
| 10 | ISP-2 → Internet | Routing | To Google server |
| 11 | Internet → ISP-2 | Return SYN-ACK | dst=198.51.100.2:49821 |
| 12 | ether2 | Receive | Connection table: ESTABLISHED, WAN2-conn |
| 13 | NAT reverse | De-translate | dst=192.168.1.50:49821 |
| 14 | Routing | Connected route | 192.168.1.0/24 via ether5 |
| 15 | ether5 | Deliver | To client MAC |
| 16 | Client | TCP established | HTTPS session active via ISP-2 |

---

## 4.7 Failover Scenario in This Design

```
NORMAL STATE:
  ISP-1: ACTIVE (distance=1, PCC bucket 0)
  ISP-2: ACTIVE (distance=1, PCC bucket 1)
  ISP-3: ACTIVE (distance=1, PCC bucket 2)
  Traffic distributed 33/33/33 via PCC

ISP-1 FAILS:
  check-gateway: 203.0.113.1 UNREACHABLE
  to-WAN1 route: INACTIVE
  PCC bucket 0 connections: DROPPED
  New connections: distributed 50/50 between ISP-2 and ISP-3
  Netwatch script: disable WAN1 mangle rules (optional)

ISP-1 RECOVERS:
  check-gateway: 203.0.113.1 REACHABLE
  to-WAN1 route: ACTIVE
  New connections: distributed 33/33/33 again
  Existing sessions on ISP-2/3: unaffected
```

---

**Next Chapter →** [Chapter 5: MikroTik Configuration](../05-mikrotik-configuration/README.md)
