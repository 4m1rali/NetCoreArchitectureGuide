# Chapter 1 — Network Architecture

> Deep engineering foundation for Multi-WAN MikroTik deployments.

---

## 1.1 What Is Multi-WAN Architecture?

**Multi-WAN Architecture** is a network design pattern where a single edge router maintains simultaneous connections to two or more independent upstream providers (ISPs) and intelligently distributes or fails over traffic between them.

### Enterprise Definition

At the ISP and Enterprise level, Multi-WAN is not simply "two internet connections." It is a **controlled egress path selection system** that manages:

- **Path diversity** — Multiple independent routes to the internet
- **Bandwidth aggregation** — Utilizing combined capacity of all WAN links
- **Resilience** — Automatic failover when a path degrades or fails
- **Policy control** — Directing specific traffic types to specific WAN paths

### Architecture Layers

```
┌─────────────────────────────────────────────────────────┐
│                    APPLICATION LAYER                     │
│              (HTTP, DNS, VoIP, VPN, etc.)               │
├─────────────────────────────────────────────────────────┤
│                    TRANSPORT LAYER (L4)                  │
│         TCP/UDP — Connection Tracking lives here         │
├─────────────────────────────────────────────────────────┤
│                    NETWORK LAYER (L3)                    │
│    IP Routing · NAT · Firewall · Mangle · PCC/ECMP      │
├─────────────────────────────────────────────────────────┤
│                   DATA LINK LAYER (L2)                   │
│         Ethernet · VLAN · PPPoE · Bridge                │
├─────────────────────────────────────────────────────────┤
│                    PHYSICAL LAYER (L1)                   │
│              Fiber · Copper · SFP · LTE                  │
└─────────────────────────────────────────────────────────┘
```

---

## 1.2 The Role of NAT in Real Networks

**Network Address Translation (NAT)** is the mechanism that maps private internal addresses to public WAN addresses. In Multi-WAN environments, NAT is not optional — it is the binding layer between internal sessions and external paths.

### Why NAT Is Critical in Multi-WAN

| Function | Without NAT | With NAT |
|----------|-------------|----------|
| Private IP egress | Impossible on public internet | Enabled via masquerade/src-nat |
| Return path consistency | Undefined | Connection tracking ensures symmetric routing |
| Per-WAN address mapping | Not possible | Each WAN can have its own public IP pool |
| Session stickiness | Broken | NAT + connection mark = stable sessions |

### NAT Types in MikroTik Multi-WAN

| Type | Command | Use Case |
|------|---------|----------|
| Masquerade | `action=masquerade` | Dynamic public IP from ISP (most common) |
| Src-NAT | `action=src-nat to-addresses=X` | Fixed public IP per WAN |
| Dst-NAT | `action=dst-nat` | Port forwarding from specific WAN |

### NAT Flow in Multi-WAN

```
Client (192.168.1.50) → Router LAN
    ↓
Routing decision: WAN2 selected (PCC mark)
    ↓
NAT: 192.168.1.50 → 203.0.113.45 (ISP-2 public IP)
    ↓
Egress via ether2 → ISP-2
    ↓
Return: 203.0.113.45 → 192.168.1.50 (connection tracking table)
```

**Critical rule:** Return traffic MUST enter the same WAN interface that carried the outbound packet. Connection tracking enforces this.

---

## 1.3 How the Routing Table Makes Decisions

The MikroTik routing table is the decision engine for every packet. In Multi-WAN, you typically operate with **multiple routing tables** beyond the default `main` table.

### Routing Decision Process

```
Packet arrives at router
    ↓
Is there a routing-mark on the packet/connection?
    ├── YES → Use the routing table associated with that mark
    └── NO  → Use main routing table
         ↓
    Find best matching route (longest prefix match)
         ↓
    Multiple equal-cost routes?
         ├── YES → ECMP (hash-based distribution)
         └── NO  → Single next-hop selected
              ↓
    Is gateway reachable? (check-gateway)
         ├── YES → Forward packet
         └── NO  → Mark route as inactive, try next route
```

### Routing Tables in Multi-WAN

| Table Name | Purpose | Created By |
|------------|---------|------------|
| `main` | Default routes, failover gateway | Static routes, DHCP |
| `to-WAN1` | PCC traffic for ISP-1 | Mangle + routing rules |
| `to-WAN2` | PCC traffic for ISP-2 | Mangle + routing rules |
| `to-WAN3` | PCC traffic for ISP-3 | Mangle + routing rules |

### Route Selection Priority

1. **Routing mark** (highest priority — overrides main table)
2. **Scope and target-scope** (recursive routing resolution)
3. **Distance** (administrative distance — lower = preferred)
4. **Gateway reachability** (check-gateway ping/interface)

---

## 1.4 Connection Tracking — Why It Is Vital

**Connection Tracking (conntrack)** is MikroTik's stateful inspection engine. It maintains a table of all active connections and their metadata.

### What Connection Tracking Stores

| Field | Purpose |
|-------|---------|
| `src-address` / `dst-address` | Original endpoints |
| `src-address` / `dst-address` (after NAT) | Translated endpoints |
| `connection-mark` | PCC routing mark |
| `routing-mark` | Active routing table selector |
| `connection-state` | new / established / related / invalid |
| `timeout` | Session lifetime |
| `protocol` | TCP / UDP / ICMP |

### Why Connection Tracking Is the Foundation of Multi-WAN

Without connection tracking:

- **PCC fails** — Marks cannot persist across packets in a session
- **NAT breaks** — Return traffic cannot be de-translated correctly
- **Asymmetric routing occurs** — Outbound via WAN1, return via WAN2
- **Firewall state rules fail** — `connection-state=established` becomes unreliable

### Connection Tracking Flow

```
Packet 1 (SYN) → NEW connection
    → Mangle: assign connection-mark "WAN2-conn"
    → NAT: translate source
    → Route via to-WAN2 table
    → Store in connection table

Packet 2 (ACK) → ESTABLISHED connection
    → Read connection-mark from table (no re-classification)
    → NAT: use stored translation
    → Route via to-WAN2 table (same path)

Packet N (FIN) → Connection teardown
    → Remove from connection table after timeout
```

---

## 1.5 L2 vs L3 vs L4 in Multi-WAN Context

Understanding which OSI layer each Multi-WAN method operates at is essential for correct design.

### Layer 2 (Data Link)

| Element | Role in Multi-WAN |
|---------|-------------------|
| Ethernet interfaces | Physical WAN/LAN attachment |
| VLANs | Separating ISP handoff from internal network |
| Bridge | LAN switching (not routing) |
| PPPoE | ISP authentication and L2 tunnel |

**L2 does NOT make routing decisions.** It delivers frames to the router's L3 engine.

### Layer 3 (Network)

| Element | Role in Multi-WAN |
|---------|-------------------|
| IP Routing | Path selection to destinations |
| ECMP | Equal-cost multipath at routing table |
| Static routes | Default gateway per WAN |
| Recursive routing | Resolving gateway reachability |
| Routing marks | Directing traffic to specific tables |
| NAT | Address translation |

**L3 is where all Multi-WAN routing decisions happen.**

### Layer 4 (Transport)

| Element | Role in Multi-WAN |
|---------|-------------------|
| Connection Tracking | Session state persistence |
| PCC (Mangle) | Per-connection classification using src+dst IP+port |
| Firewall | Stateful filtering by connection |
| NAT port mapping | TCP/UDP port translation |

**L4 provides session awareness that makes PCC and stateful NAT possible.**

### Layer Interaction Diagram

```
[L2: Frame arrives on ether1]
        ↓
[L3: IP header processed, routing table consulted]
        ↓
[L4: Connection tracking lookup/creation]
        ↓
[L4: Mangle applies connection-mark (PCC)]
        ↓
[L3: Routing mark set, specific table selected]
        ↓
[L3/L4: NAT translation applied]
        ↓
[L3: Forwarding decision → egress interface]
        ↓
[L2: Frame transmitted on WAN interface]
```

---

## 1.6 Real Packet Journey Through the Router

### Complete Step-by-Step Flow

**Scenario:** Client `192.168.1.100` requests `https://example.com` (203.0.113.50) via a 3-WAN PCC setup.

```
STEP 1 — INGRESS (LAN → Router)
─────────────────────────────────
Interface: ether5 (LAN bridge)
Source: 192.168.1.100:52431
Destination: 203.0.113.50:443
Protocol: TCP SYN

STEP 2 — FIREWALL: INPUT CHAIN
─────────────────────────────────
Rule: accept established,related → SKIP (new connection)
Rule: accept from LAN → PASS

STEP 3 — FIREWALL: FORWARD CHAIN
─────────────────────────────────
Rule: accept established,related → SKIP (new)
Rule: fasttrack → SKIP (new connection, no fasttrack yet)
Rule: accept from LAN to WAN → PASS

STEP 4 — MANGLE: PREROUTING
─────────────────────────────────
Rule: PCC classifier
  → connection-mark: WAN2-conn (hash of src+dst → ISP-2)
  → routing-mark: to-WAN2

STEP 5 — CONNECTION TRACKING
─────────────────────────────────
Action: CREATE new connection entry
  → Store: src=192.168.1.100:52431, dst=203.0.113.50:443
  → Store: connection-mark=WAN2-conn
  → State: new → established (after handshake)

STEP 6 — ROUTING DECISION
─────────────────────────────────
Routing-mark: to-WAN2 → use table "to-WAN2"
Route: 0.0.0.0/0 via 198.51.100.1 (ISP-2 gateway) → ACTIVE
Next-hop interface: ether2

STEP 7 — NAT: SRCNAT
─────────────────────────────────
Rule: masquerade out-interface=ether2
  → 192.168.1.100:52431 → 198.51.100.45:52431

STEP 8 — FIREWALL: FORWARD (POST-NAT)
─────────────────────────────────
Rule: accept to WAN → PASS

STEP 9 — EGRESS
─────────────────────────────────
Interface: ether2
Source: 198.51.100.45:52431
Destination: 203.0.113.50:443
→ Transmitted to ISP-2

STEP 10 — RETURN PATH
─────────────────────────────────
Interface: ether2 (MUST be same WAN)
Destination: 198.51.100.45:52431
→ Connection table lookup → ESTABLISHED
→ routing-mark: to-WAN2 (from connection table)
→ NAT reverse: 198.51.100.45 → 192.168.1.100
→ Forward to ether5 (LAN)
```

---

## 1.7 Architecture Principles

| Principle | Description |
|-----------|-------------|
| **Symmetric routing** | Outbound and return traffic must use the same WAN path |
| **Connection stickiness** | Once a session is classified, it stays on that path |
| **Mark before route** | Mangle (PCC) must run before routing decision |
| **NAT after route mark** | NAT rules should reference out-interface or connection-mark |
| **Monitor all gateways** | Every WAN gateway must have check-gateway enabled |
| **Separate routing tables** | One table per WAN for PCC; main table for failover |

---

## 1.8 VRF and Routing Table Isolation

**VRF (Virtual Routing and Forwarding)** isolates routing tables so different traffic domains never leak routes between each other. In RouterOS 7, VRF is implemented via `/routing table` with `fib` flag and optional VRF interface binding.

### When VRF Matters in Multi-WAN

| Scenario | VRF Use |
|----------|---------|
| ISP wholesale + retail on same router | Separate customer routing from management |
| MPLS + Internet on same CCR | Internet Multi-WAN in VRF-A, MPLS in VRF-B |
| Guest + Corporate network | Guest traffic forced through filtered WAN |
| Management plane isolation | Out-of-band management never uses customer WAN tables |

### VRF + PCC Architecture

```
VRF: internet-vrf
  ├── ether1 (ISP-1) → to-WAN1 table
  ├── ether2 (ISP-2) → to-WAN2 table
  ├── ether3 (ISP-3) → to-WAN3 table
  └── ether5 (LAN)   → PCC mangle applies here

VRF: mgmt-vrf
  └── ether6 (Management) → single dedicated WAN or VPN
```

### Key Rule

PCC mangle, routing tables, and NAT must all exist **inside the same VRF/FIB context**. A connection mark in one VRF must never reference routes in another.

---

## 1.9 MTU, MSS, and Fragmentation in Multi-WAN

Different ISPs often have different MTU values. This causes silent failures in Multi-WAN deployments.

| ISP Type | Typical MTU | Notes |
|----------|-------------|-------|
| Fiber / Ethernet | 1500 | Standard |
| PPPoE | 1492 | 8-byte PPPoE header |
| GRE / VPN overlay | 1400–1476 | Depends on encapsulation |
| LTE | 1400–1428 | Carrier-dependent |

### MSS Clamping (Mandatory for PPPoE/LTE WAN)

```
/ip firewall mangle
add chain=forward protocol=tcp tcp-flags=syn action=change-mss new-mss=1440 passthrough=yes \
    out-interface-list=WAN comment="MSS clamp WAN"
```

### MTU Mismatch Symptoms

| Symptom | Cause |
|---------|-------|
| Small packets work, large downloads fail | MTU black hole |
| HTTPS fails, ping works | PMTUD blocked, MSS not clamped |
| One WAN works, another fails same site | Different ISP MTU |
| VPN connects but no traffic | Tunnel MTU too high |

### Per-Interface MTU Setting

```
/interface ethernet
set ether1 mtu=1500
set ether2 mtu=1492 comment="PPPoE ISP-2"
set lte1 mtu=1420 comment="LTE ISP-3"
```

---

## 1.10 Bridge Mode vs Router Mode on WAN

| Mode | Use | Multi-WAN Impact |
|------|-----|-----------------|
| **Router mode** (recommended) | Each WAN is a routed interface | Full PCC, NAT, firewall control |
| **Bridge mode** | WAN bridged to LAN (transparent) | Cannot route between WANs — avoid |
| **Hybrid** | WAN routed, LAN bridged | Most common production pattern |

### Recommended LAN Design

```
/interface bridge
add name=bridge-lan vlan-filtering=no

/interface bridge port
add bridge=bridge-lan interface=ether5
add bridge=bridge-lan interface=ether6

/interface list member
add interface=bridge-lan list=LAN
```

WAN interfaces (ether1–3) must **never** be bridge ports in Multi-WAN designs.

---

## 1.11 FastTrack vs Connection Tracking in Multi-WAN

**FastTrack** bypasses connection tracking and mangle for performance. In PCC deployments, FastTrack **breaks** load balancing.

| Setting | PCC Compatible | Performance |
|---------|---------------|-------------|
| FastTrack enabled | **NO** | Highest throughput |
| FastTrack disabled | **YES** | Required for PCC |
| FastTrack + HW offload | **NO** | Bypasses mangle entirely |

### Rule Order Impact

```
# WRONG — FastTrack before PCC awareness
add chain=forward action=fasttrack-connection connection-state=established,related

# CORRECT for PCC — no FastTrack on balanced traffic
add chain=forward action=accept connection-state=established,related
```

For CCR hardware at scale without PCC (failover-only), FastTrack is acceptable.

---

## 1.12 Connection State Machine

```
                    ┌──────────┐
         ┌─────────│   NEW    │─────────┐
         │         └────┬─────┘         │
         │              │               │
    (invalid)      (SYN-ACK)         (RST)
         │              │               │
         ▼              ▼               ▼
    ┌─────────┐  ┌─────────────┐  ┌─────────┐
    │ INVALID │  │ ESTABLISHED │  │  DROP   │
    └─────────┘  └──────┬──────┘  └─────────┘
                        │
                   (FIN / timeout)
                        │
                        ▼
                 ┌─────────────┐
                 │  TIME-WAIT  │
                 └─────────────┘
```

| State | Multi-WAN Behavior |
|-------|-------------------|
| `new` | PCC classifier runs — connection assigned to WAN |
| `established` | Mark read from table — no re-classification |
| `related` | FTP/data channel follows parent connection mark |
| `invalid` | Drop — often indicates asymmetric routing or NAT failure |
| `untracked` | Bypasses conntrack — policy routing must handle manually |

---

## 1.13 ISP Handoff Types

| Handoff | Configuration | Multi-WAN Notes |
|---------|--------------|-----------------|
| Static IP | `/ip address` + static route | Simplest — reference design |
| DHCP | `/ip dhcp-client` | Gateway may change — use script to update routes |
| PPPoE | `/interface pppoe-client` | Use interface as gateway, MTU 1492 |
| BGP | `/routing bgp` | Datacenter/ISP — see Chapter 13 |
| LTE | `/interface lte` | Dynamic IP — monitor with Netwatch |
| VLAN from ISP | `/interface vlan` | Tag per ISP on single physical port |

---

**Next Chapter →** [Chapter 2: Core Concepts](../02-core-concepts/README.md)
