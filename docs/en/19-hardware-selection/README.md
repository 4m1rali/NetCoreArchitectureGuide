# Chapter 19 — Hardware Selection Guide

> Choosing the right MikroTik hardware for Multi-WAN production deployments.

---

## 19.1 Hardware Selection Matrix

| Model | CPU | RAM | Ports | Throughput | Max Users | Multi-WAN Role |
|-------|-----|-----|-------|------------|-----------|----------------|
| hEX S (RB760iGS) | MMIPS 880MHz | 256MB | 5xGE + SFP | ~400Mbps | 25 | Branch failover |
| RB750Gr3 | MMIPS 880MHz | 256MB | 5xGE | ~400Mbps | 25 | SOHO dual-WAN |
| RB4011iGS+RM | IPQ-4019 4-core | 1GB | 10xGE + SFP+ | ~3.5Gbps | 200 | **Enterprise standard** |
| CCR2004-16G-2S+ | AL32400 4-core | 4GB | 16xGE + 2xSFP+ | ~10Gbps | 500 | ISP edge |
| CCR2116-12G-4S+ | AL73400 16-core | 16GB | 12xGE + 4xSFP+ | ~25Gbps | 2000 | ISP core |
| CCR2216-1G-12XS-2XQ | AL73400 16-core | 16GB | 1xGE + 12xSFP+ + 2xQSFP | ~100Gbps | 10000+ | Datacenter |

---

## 19.2 Selection by Scenario

### SOHO / Home Office (2 WAN, < 10 users)

| Item | Recommendation |
|------|---------------|
| Model | hEX S or RB750Gr3 |
| Method | Failover only |
| License | Level 4 |
| Estimated cost | $60–80 |

### SMB / Branch (2–3 WAN, 50 users)

| Item | Recommendation |
|------|---------------|
| Model | RB4011iGS+RM |
| Method | PCC 2-way + Failover |
| License | Level 4–5 |
| Estimated cost | $250–300 |

### Enterprise HQ (3 WAN, 200 users)

| Item | Recommendation |
|------|---------------|
| Model | RB4011 (dual) or CCR2004 |
| Method | PCC + Policy + QoS |
| License | Level 5 |
| Estimated cost | $300–800 |

### ISP WISP (3 WAN, 300+ subscribers)

| Item | Recommendation |
|------|---------------|
| Model | CCR2004 or CCR2116 |
| Method | PCC + PCQ + BGP |
| License | Level 5–6 |
| Estimated cost | $800–2500 |

### Datacenter Edge (BGP, 10G+)

| Item | Recommendation |
|------|---------------|
| Model | CCR2116 or CCR2216 |
| Method | BGP + ECMP |
| License | Level 6 |
| Estimated cost | $2500–5000 |

---

## 19.3 CPU and PCC Performance

PCC mangle rules consume CPU per **new connection**. Hardware selection must account for connection rate, not just bandwidth.

| Metric | hEX | RB4011 | CCR2004 | CCR2116 |
|--------|-----|--------|---------|---------|
| New conn/sec | ~500 | ~5,000 | ~20,000 | ~80,000 |
| Active connections | 50K | 200K | 500K | 2M+ |
| PCC 3-WAN CPU @ 1Gbps | 60–80% | 15–25% | 5–10% | < 5% |
| Recommended PCC WANs | 2 | 3 | 3–4 | 4+ |

**Rule:** If sustained new connections exceed 2000/sec, use CCR series.

---

## 19.4 RAM Requirements

| Component | RAM Usage |
|-----------|-----------|
| RouterOS base | ~50MB |
| Connection tracking (100K entries) | ~200MB |
| BGP full table | ~2–4GB |
| BGP default only | ~50MB |
| PCC mangle (no extra RAM) | CPU only |
| User Manager (500 users) | ~100MB |

| Deployment | Minimum RAM |
|------------|-------------|
| Basic Multi-WAN | 256MB |
| Enterprise PCC | 1GB |
| ISP with 500K connections | 4GB |
| BGP full table | 8–16GB |

---

## 19.5 Storage and Logging

| Model | Storage | Logging Capacity |
|-------|---------|-----------------|
| hEX S | 16MB NAND | Minimal — use remote syslog |
| RB4011 | 512MB NAND | Moderate |
| CCR2004 | 128MB NAND + USB | Good with USB disk |
| CCR2116 | 128MB NAND + NVMe slot | Excellent |

For production Multi-WAN, always use **remote Syslog** regardless of model.

---

## 19.6 Interface Count Planning

| WAN Type | Interfaces Needed |
|----------|------------------|
| 2 ISP static fiber | 2 WAN + 1 LAN = 3 |
| 3 ISP mixed | 3 WAN + 1 LAN + 1 mgmt = 5 |
| ISP with VLAN handoff | 1 physical + VLANs |
| PPPoE × 3 | 3 physical WAN ports |
| Dual router VRRP | 2× everything |

**Minimum:** 1 LAN + N WAN + 1 management port

---

## 19.7 Redundant Power and Environment

| Requirement | Enterprise | ISP |
|-------------|-----------|-----|
| Dual PSU | Recommended | Mandatory |
| Rack mount | RB4011 1U or CCR | CCR 1U |
| Temperature | < 40°C ambient | < 35°C with airflow |
| UPS | Mandatory | Mandatory with generator |
| Out-of-band mgmt | Dedicated mgmt port | Serial + dedicated mgmt |

---

## 19.8 Hardware Checklist

| # | Item | ☐ |
|---|------|---|
| 1 | Enough WAN ports for all ISPs | ☐ |
| 2 | SFP/SFP+ if fiber WAN required | ☐ |
| 3 | RAM sufficient for connection count | ☐ |
| 4 | CPU sufficient for PCC at target bandwidth | ☐ |
| 5 | License level matches feature requirements | ☐ |
| 6 | Storage/USB for logging (or remote syslog planned) | ☐ |
| 7 | Rack mount kit if datacenter deployment | ☐ |
| 8 | UPS capacity calculated for router power draw | ☐ |

---

**Next Chapter →** [Chapter 20: Migration & Upgrade](../20-migration-upgrade/README.md)
