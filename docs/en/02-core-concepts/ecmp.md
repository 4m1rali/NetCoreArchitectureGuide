# ECMP Routing

> Equal-Cost Multi-Path — L3 routing-level load distribution.

---

## Engineering Definition

**ECMP (Equal-Cost Multi-Path)** is a routing mechanism where multiple next-hop paths to the same destination prefix have identical metric (distance) values, causing the router to distribute traffic across all available paths simultaneously.

At the enterprise level, ECMP operates purely at **Layer 3** — it hashes each packet (or flow, depending on implementation) to one of several equal-cost gateways without session awareness.

---

## Internal Router Flow (Step-by-Step)

```
1. Packet arrives with destination 0.0.0.0/0 (default route)
2. Routing table lookup in "main" table
3. Multiple routes found with SAME distance:
   → 0.0.0.0/0 via 203.0.113.1 distance=1
   → 0.0.0.0/0 via 198.51.100.1 distance=1
   → 0.0.0.0/0 via 192.0.2.1 distance=1
4. ECMP hash algorithm applied:
   → Input: src-IP, dst-IP, protocol, src-port, dst-port
   → Output: selected gateway index (0, 1, or 2)
5. Packet forwarded via selected gateway interface
6. NEXT packet in same TCP session MAY use different gateway
   (no connection tracking involvement in pure ECMP)
```

---

## Behavior in MikroTik RouterOS

### Configuration Pattern

```
/ip route
add dst-address=0.0.0.0/0 gateway=203.0.113.1 distance=1 check-gateway=ping
add dst-address=0.0.0.0/0 gateway=198.51.100.1 distance=1 check-gateway=ping
add dst-address=0.0.0.0/0 gateway=192.0.2.1 distance=1 check-gateway=ping
```

### MikroTik-Specific Behavior

| Behavior | Detail |
|----------|--------|
| Hash input | Layer 3/4 header fields |
| Distribution model | Per-packet (default) or per-connection (with ECMP per-connection-setting) |
| Route removal | When check-gateway fails, route becomes inactive — traffic redistributed |
| Table scope | Operates in single routing table only |
| NAT interaction | Problematic — sessions may break due to per-packet path changes |

### ECMP Per-Connection Mode (RouterOS 7+)

```
/routing settings
set ecmp-per-connection=yes
```

This makes ECMP behave more like flow-based distribution, improving NAT compatibility.

---

## Use Cases

| Environment | Application |
|-------------|-------------|
| ISP Core | Multiple upstream peer links with BGP |
| Datacenter | Equal-cost paths to internet exchange |
| Enterprise (no NAT) | Routed public IP blocks across WANs |
| Lab/Testing | Simple bandwidth aggregation without PCC complexity |

---

## Pros

| Advantage | Detail |
|-----------|--------|
| Simplicity | Minimal configuration — just add routes with same distance |
| Performance | No mangle rules, no connection marks — lowest CPU overhead |
| Scalability | Easily add more WANs by adding routes |
| Fast convergence | check-gateway removes dead routes quickly |
| Native routing | Uses standard routing table — no custom tables needed |

---

## Cons and Risks

| Risk | Detail |
|------|--------|
| NAT breakage | Per-packet hashing breaks connection tracking symmetry |
| Session instability | HTTPS, VPN, VoIP sessions may reset mid-connection |
| No traffic policy | Cannot direct specific traffic to specific WAN |
| Asymmetric routing | Return path may differ if upstream ISPs have different routing |
| No bandwidth weighting | All paths treated equally regardless of capacity |
| Firewall state issues | `connection-state=established` may fail across path changes |

---

## Common Implementation Errors

| Error | Consequence | Fix |
|-------|-------------|-----|
| ECMP with masquerade NAT | Broken sessions, intermittent connectivity | Use PCC instead |
| Missing check-gateway | Dead routes remain active, blackhole traffic | Add `check-gateway=ping` to all routes |
| Different distances | Only one route used, no load balancing | Ensure all routes have identical distance |
| ECMP + PCC simultaneously | Conflicting routing decisions | Choose one method per traffic class |
| No firewall state rules | Invalid connection states | Add established/related accept rules first |

---

**Next →** [PCC (Per Connection Classifier)](pcc.md)
