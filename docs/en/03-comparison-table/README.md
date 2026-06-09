# Chapter 3 — Enterprise Comparison Table

> Professional method comparison matrix for Multi-WAN deployment decisions.

---

## Multi-WAN Method Comparison Matrix

| Attribute | ECMP | PCC | Failover | ECMP + Failover | PCC + Failover |
|-----------|------|-----|----------|-----------------|----------------|
| **Method Type** | Routing | Mangle + Routing | Route Priority | Combined | Combined |
| **OSI Layer** | L3 | L3/L4 | L3 | L3 | L3/L4 |
| **Load Distribution Model** | Packet / Flow | Flow (Per-Connection) | None (Active/Standby) | Flow | Flow |
| **Session Stability** | Poor (per-packet) / Good (per-conn) | Excellent | N/A (single path) | Good | Excellent |
| **NAT Compatibility** | Poor | Excellent | Good | Moderate | Excellent |
| **Performance Impact** | Minimal | Low–Moderate | Minimal | Low | Low–Moderate |
| **CPU Usage** | Very Low | Moderate | Very Low | Low | Moderate |
| **Scalability** | High (add routes) | High (add buckets) | High | High | High |
| **Complexity Level** | Low | High | Low | Medium | High |
| **Failure Risk** | Medium (NAT breaks) | Low | Very Low | Low | Very Low |
| **Best Use Case** | Routed public IPs, no NAT | NAT environments, ISP/Enterprise | Backup WAN, critical uptime | Datacenter, BGP | **Production Multi-WAN (Recommended)** |

---

## Detailed Attribute Analysis

### Load Distribution Model

| Method | Model | Explanation |
|--------|-------|-------------|
| ECMP (default) | Per-Packet | Each packet individually hashed — fastest but breaks NAT |
| ECMP (per-conn) | Per-Flow | Each connection hashed once — better NAT compatibility |
| PCC | Per-Flow | Connection mark persists for session lifetime |
| Failover | Active/Standby | All traffic on primary until failure |

### Session Stability Rating

| Method | Rating | Detail |
|--------|--------|--------|
| ECMP per-packet | ★☆☆☆☆ | Sessions break constantly with NAT |
| ECMP per-connection | ★★★☆☆ | Stable within connection, no per-WAN NAT control |
| PCC | ★★★★★ | Full session stickiness with per-WAN NAT |
| Failover | ★★★★☆ | Stable until failover event (sessions drop once) |

### NAT Compatibility Rating

| Method | Rating | Detail |
|--------|--------|--------|
| ECMP per-packet | ★☆☆☆☆ | Masquerade breaks return path |
| ECMP per-connection | ★★★☆☆ | Works with single masquerade rule |
| PCC | ★★★★★ | Per-interface masquerade with connection marks |
| Failover | ★★★★☆ | Single active NAT rule sufficient |

### CPU Usage Comparison

| Method | New Connection Cost | Per-Packet Cost | 10K Connections |
|--------|--------------------|-----------------|-----------------| 
| ECMP | None | Hash only | ~0% CPU |
| PCC | Mangle evaluation | Mark lookup | ~5–15% CPU |
| Failover | Ping only | None | ~1% CPU |
| PCC + Failover | Mangle + ping | Mark lookup | ~5–20% CPU |

### Complexity Level

| Method | Config Items | Routing Tables | Mangle Rules | NAT Rules |
|--------|-------------|----------------|--------------|-----------|
| ECMP | 3–5 | 1 (main) | 0 | 1 |
| PCC | 15–25 | 4 (main + 3 WAN) | 6–9 | 3 |
| Failover | 3–5 | 1 (main) | 0 | 1 |
| PCC + Failover | 20–30 | 4 | 6–9 | 3 |

### Failure Risk Assessment

| Method | Risk Level | Primary Risk |
|--------|-----------|--------------|
| ECMP + NAT | **HIGH** | Session breakage, asymmetric routing |
| PCC | **LOW** | Misconfigured mangle order |
| Failover only | **VERY LOW** | Single point of bandwidth |
| PCC + Failover | **VERY LOW** | PCC orphan on failed WAN (mitigated with Netwatch) |

---

## Decision Matrix

| Your Requirement | Recommended Method |
|-----------------|-------------------|
| NAT + Load Balancing + Failover | **PCC + Failover** |
| Public IP routing, no NAT | **ECMP + Failover** |
| Maximum uptime, single WAN sufficient | **Failover only** |
| Simple lab / testing | **ECMP** |
| ISP with 3 upstreams | **PCC + Failover + Per-WAN NAT** |
| Enterprise 100+ users | **PCC + Failover** |
| VoIP + Data mixed traffic | **PCC + Policy Routing + Failover** |
| Datacenter BGP multi-homed | **BGP + ECMP** |

---

## Feature Support Matrix (MikroTik RouterOS)

| Feature | ECMP | PCC | Failover |
|---------|------|-----|----------|
| RouterOS 6.x | Yes | Yes | Yes |
| RouterOS 7.x | Yes (enhanced) | Yes | Yes |
| IPv6 | Yes | Yes | Yes |
| VRF | Yes | Yes | Yes |
| FastTrack compatible | Yes | Partial (bypasses mangle) | Yes |
| Hardware offload | Yes | No (mangle) | Yes |
| VLAN interfaces | Yes | Yes | Yes |
| PPPoE WAN | Yes | Yes | Yes |
| Bridge WAN | Not recommended | Not recommended | Yes |

---

**Next Chapter →** [Chapter 4: Production Design](../04-production-design/README.md)
