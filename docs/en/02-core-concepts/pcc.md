# PCC (Per Connection Classifier)

> The industry-standard method for Multi-WAN load balancing on MikroTik.

---

## Engineering Definition

**PCC (Per Connection Classifier)** is a traffic classification technique that uses a deterministic hash of connection parameters (source IP, destination IP, source port, destination port, protocol) to assign each new connection to a specific WAN path. Once classified, the connection retains its assigned path for its entire lifetime via connection marks.

PCC operates at **Layer 3/4** through MikroTik's mangle firewall and routing-mark system.

---

## Internal Router Flow (Step-by-Step)

```
1. NEW packet arrives (no connection table entry)
2. Mangle PREROUTING chain:
   a. Check: connection-mark is empty (new connection)
   b. Calculate PCC hash:
      hash = (src-ip XOR dst-ip XOR src-port XOR dst-port) MOD N
      where N = number of WAN links
   c. Result 0 → assign connection-mark "WAN1-conn"
      Result 1 → assign connection-mark "WAN2-conn"
      Result 2 → assign connection-mark "WAN3-conn"
   d. Set routing-mark based on connection-mark
3. Connection tracking: STORE connection-mark
4. Routing: use routing table matching routing-mark
   → "to-WAN1" / "to-WAN2" / "to-WAN3"
5. NAT: masquerade on correct out-interface
6. SUBSEQUENT packets (ESTABLISHED):
   a. Connection table provides stored connection-mark
   b. Mangle rules SKIPPED (mark already set)
   c. Same routing table, same NAT, same WAN path
7. Connection closes → entry removed from table
```

---

## Behavior in MikroTik RouterOS

### PCC Mangle Rule Pattern

```
/ip firewall mangle
add chain=prerouting dst-address-type=!local in-interface-list=LAN \
    per-connection-classifier=both-addresses-and-ports:3/0 \
    action=mark-connection new-connection-mark=WAN1-conn passthrough=yes

add chain=prerouting dst-address-type=!local in-interface-list=LAN \
    per-connection-classifier=both-addresses-and-ports:3/1 \
    action=mark-connection new-connection-mark=WAN2-conn passthrough=yes

add chain=prerouting dst-address-type=!local in-interface-list=LAN \
    per-connection-classifier=both-addresses-and-ports:3/2 \
    action=mark-connection new-connection-mark=WAN3-conn passthrough=yes
```

### Classifier Syntax

```
per-connection-classifier=both-addresses-and-ports:N/M
```

| Parameter | Meaning |
|-----------|---------|
| `both-addresses-and-ports` | Hash input: src-ip + dst-ip + src-port + dst-port |
| `N` | Total number of WAN links |
| `M` | Bucket index (0 to N-1) |

### Routing Mark Assignment

```
add chain=prerouting connection-mark=WAN1-conn in-interface-list=LAN \
    action=mark-routing new-routing-mark=to-WAN1 passthrough=yes

add chain=prerouting connection-mark=WAN2-conn in-interface-list=LAN \
    action=mark-routing new-routing-mark=to-WAN2 passthrough=yes

add chain=prerouting connection-mark=WAN3-conn in-interface-list=LAN \
    action=mark-routing new-routing-mark=to-WAN3 passthrough=yes
```

### Routing Tables

```
/ip route
add dst-address=0.0.0.0/0 gateway=203.0.113.1 routing-table=to-WAN1 check-gateway=ping
add dst-address=0.0.0.0/0 gateway=198.51.100.1 routing-table=to-WAN2 check-gateway=ping
add dst-address=0.0.0.0/0 gateway=192.0.2.1 routing-table=to-WAN3 check-gateway=ping
```

---

## Use Cases

| Environment | Why PCC |
|-------------|---------|
| ISP Edge Router | Distribute subscriber traffic across upstreams |
| Enterprise Branch | Session-stable load balancing for 50–500 users |
| SOHO with 2 ISPs | Reliable dual-WAN without session drops |
| VoIP + Data mixed | Each call stays on one path (no mid-call switching) |
| Gaming / Streaming | Stable latency per session |

---

## Pros

| Advantage | Detail |
|-----------|--------|
| Session stability | Connection mark persists for entire session lifetime |
| NAT compatible | Works correctly with masquerade and src-nat |
| Predictable distribution | Hash algorithm gives ~equal distribution over time |
| Per-WAN policy | Each WAN has its own routing table and NAT rules |
| Production proven | Most widely deployed Multi-WAN method on MikroTik |
| Firewall compatible | Stateful rules work correctly with connection marks |

---

## Cons and Risks

| Risk | Detail |
|------|--------|
| CPU overhead | Mangle rules process every new connection |
| Configuration complexity | Requires mangle + routing tables + NAT coordination |
| Uneven real-time distribution | Short-lived connections may cluster on one WAN |
| No bandwidth weighting | 3/0, 3/1, 3/2 gives equal split (not capacity-aware) |
| Rule ordering critical | Incorrect mangle order breaks classification |
| Connection table size | High connection count increases memory usage |

---

## Common Implementation Errors

| Error | Consequence | Fix |
|-------|-------------|-----|
| Missing `passthrough=yes` | Only first packet classified, rest unmarked | Always set passthrough=yes on connection marks |
| Mangle after routing | Routing decision made before classification | Mangle must be in prerouting, before routing |
| No separate routing tables | All traffic uses main table | Create to-WAN1, to-WAN2, to-WAN3 tables |
| NAT without out-interface filter | Wrong WAN IP used for translation | Match NAT rules to specific out-interface |
| Classifier N/M mismatch | Uneven or missing distribution | N = total WANs, M = 0 to N-1 |
| Forgetting established bypass | Re-classification of active sessions | Add `connection-mark=no-mark` condition for classifier rules |
| No check-gateway on PCC routes | Traffic sent to dead WAN | Enable check-gateway on all PCC routing table routes |

---

**Next →** [Failover](failover.md)
