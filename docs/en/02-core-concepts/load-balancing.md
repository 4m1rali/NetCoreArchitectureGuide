# Load Balancing

> Conceptual framework and practical implementation for Multi-WAN traffic distribution.

---

## Engineering Definition

**Load Balancing** in Multi-WAN context is the practice of distributing outbound (and optionally inbound) network traffic across multiple WAN links to maximize bandwidth utilization, reduce congestion on individual paths, and improve overall network throughput.

Load balancing is the **objective** — PCC and ECMP are the **methods** to achieve it.

---

## Distribution Models

| Model | Layer | Granularity | Session Safe |
|-------|-------|-------------|--------------|
| Per-Packet | L3 | Individual packets | No |
| Per-Flow / Per-Connection | L3/L4 | Entire connection/session | Yes |
| Per-Route | L3 | Routing table decision | Depends |
| Weighted | L3/L4 | Capacity-proportional | Yes (with PCC) |

---

## Internal Router Flow — Load Balancing Decision Tree

```
Traffic arrives at edge router
    ↓
Is load balancing required?
    ├── NO → Single WAN (failover only)
    └── YES
         ↓
    Is NAT required?
         ├── YES → Use PCC (per-connection)
         └── NO
              ↓
         Is session stability required?
              ├── YES → Use PCC or ECMP per-connection
              └── NO → ECMP per-packet acceptable
                   ↓
              Are WAN capacities equal?
                   ├── YES → Equal PCC classifier (3/0, 3/1, 3/2)
                   └── NO → Weighted PCC or policy routing
```

---

## Practical Load Balancing with PCC

### Equal Distribution (3 WAN, 33/33/33)

```
per-connection-classifier=both-addresses-and-ports:3/0  → WAN1
per-connection-classifier=both-addresses-and-ports:3/1  → WAN2
per-connection-classifier=both-addresses-and-ports:3/2  → WAN3
```

### Weighted Distribution (2 WAN, 70/30)

For unequal capacity links, use multiple classifier buckets:

```
# WAN1 (70%) — buckets 0, 1, 2, 3, 4, 5, 6
per-connection-classifier=both-addresses-and-ports:10/0
per-connection-classifier=both-addresses-and-ports:10/1
per-connection-classifier=both-addresses-and-ports:10/2
per-connection-classifier=both-addresses-and-ports:10/3
per-connection-classifier=both-addresses-and-ports:10/4
per-connection-classifier=both-addresses-and-ports:10/5
per-connection-classifier=both-addresses-and-ports:10/6

# WAN2 (30%) — buckets 7, 8, 9
per-connection-classifier=both-addresses-and-ports:10/7
per-connection-classifier=both-addresses-and-ports:10/8
per-connection-classifier=both-addresses-and-ports:10/9
```

### Policy-Based Load Balancing

Direct specific traffic types to specific WANs:

```
# VoIP → WAN1 (low latency)
add chain=prerouting protocol=udp dst-port=5060,5061 \
    action=mark-connection new-connection-mark=WAN1-conn

# Backup traffic → WAN3 (cheaper link)
add chain=prerouting connection-bytes=10000000-0 \
    action=mark-connection new-connection-mark=WAN3-conn
```

---

## Behavior in MikroTik RouterOS

### What MikroTik Can Load Balance

| Traffic Type | Method | Notes |
|-------------|--------|-------|
| Outbound TCP | PCC | Full session stickiness |
| Outbound UDP | PCC | Works, shorter timeout |
| Outbound ICMP | ECMP or PCC | Low volume, usually irrelevant |
| Inbound (DNAT) | Policy routing | Requires per-WAN dst-nat rules |
| VPN tunnels | Policy routing | One tunnel per WAN, not balanced |

### What MikroTik Cannot Load Balance

| Traffic Type | Reason |
|-------------|--------|
| Single TCP connection | Cannot split one connection across WANs |
| Inbound initiated connections | ISP routing determines ingress path |
| Encrypted VPN inside tunnel | Outer tunnel is one connection |
| Multicast | Not routable across unequal paths |

---

## Use Cases

| Environment | Strategy |
|-------------|----------|
| ISP (500+ subscribers) | PCC 3-way + per-WAN NAT + Netwatch failover |
| Enterprise (100 users) | PCC 2-way equal + failover |
| SOHO (10 devices) | PCC 2-way or simple failover |
| Datacenter outbound | ECMP per-connection + BGP |
| Branch with VoIP | Policy routing: VoIP→WAN1, Data→PCC |

---

## Pros

| Advantage | Detail |
|-----------|--------|
| Bandwidth aggregation | Combined throughput of all WAN links |
| Congestion reduction | No single link bears all traffic |
| Cost optimization | Use cheaper links for bulk, premium for latency-sensitive |
| Resilience + distribution | Combined with failover for full solution |
| Scalable | Add WAN links by extending classifier buckets |

---

## Cons and Risks

| Risk | Detail |
|------|--------|
| Not true bonding | 3x100Mbps ≠ 300Mbps for single connection |
| Measurement difficulty | Per-WAN utilization hard to monitor without tools |
| DNS issues | Different WANs may resolve to different CDN nodes |
| Geo-IP inconsistency | External services see different public IPs |
| SSL/TLS session issues | Certificate pinning may conflict with IP changes |
| Gaming NAT type | Strict NAT on load-balanced connections |

---

## Common Implementation Errors

| Error | Consequence | Fix |
|-------|-------------|-----|
| Expecting single-stream speedup | User sees no improvement for downloads | Educate: balancing is across connections, not within |
| No per-WAN DNS | DNS resolves via wrong WAN | Set per-WAN DNS or use router DNS cache |
| Ignoring upload capacity | Download balanced but upload congested | Monitor both directions per WAN |
| Balancing without monitoring | Dead WAN still receives new connections | Combine with check-gateway + Netwatch |
| Too many mangle rules | CPU bottleneck on high-connection routers | Optimize rule count, use hardware offload |
| Missing connection-mark persistence | Every packet re-classified | Verify passthrough=yes and connection table |

---

**Next Chapter →** [Chapter 3: Comparison Table](../03-comparison-table/README.md)
