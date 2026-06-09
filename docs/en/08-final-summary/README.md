# Chapter 8 — Final Architecture Summary

> Engineering conclusions and production deployment recommendations.

---

## 8.1 Best Multi-WAN Design

### Reference Architecture

```
                    ┌──────────────────────────────┐
                    │     PRODUCTION MULTI-WAN      │
                    │                              │
   ISP-1 ──────────│  PCC Bucket 0  → to-WAN1    │
   (Primary)       │  ISP-2 ────────│  PCC Bucket 1  → to-WAN2    │
                   │  ISP-3 ────────│  PCC Bucket 2  → to-WAN3    │
   (Backup)        │                              │
                   │  Failover: check-gateway     │
                   │  NAT: per-WAN masquerade     │
                   │  Firewall: stateful + anti-  │
                   │            loop              │
                   │  Monitor: Netwatch + logs    │
                    └──────────────────────────────┘
                              │
                         LAN 192.168.1.0/24
```

### Design Principles

| # | Principle | Implementation |
|---|-----------|---------------|
| 1 | Session stickiness | PCC connection marks |
| 2 | Path redundancy | check-gateway + Netwatch |
| 3 | NAT symmetry | Per-interface masquerade |
| 4 | Security default-deny | Stateful firewall, WAN input drop |
| 5 | Observable | Mangle stats, traffic monitor, logs |
| 6 | Recoverable | Config export, documented IP plan |

---

## 8.2 Best Load Distribution Method

| Rank | Method | Score | Verdict |
|------|--------|-------|---------|
| 1 | **PCC** | ★★★★★ | **Production standard** — use this |
| 2 | ECMP per-connection | ★★★☆☆ | Acceptable without NAT only |
| 3 | ECMP per-packet | ★★☆☆☆ | Lab/testing only |
| 4 | Policy routing | ★★★☆☆ | Supplement to PCC, not replacement |
| 5 | NTH (legacy) | ★☆☆☆☆ | Deprecated — do not use |

### Distribution Strategy by WAN Count

| WANs | Classifier | Split |
|------|-----------|-------|
| 2 | `:2/0`, `:2/1` | 50% / 50% |
| 3 | `:3/0`, `:3/1`, `:3/2` | 33% / 33% / 33% |
| 4 | `:4/0` through `:4/3` | 25% each |
| 2 (70/30) | `:10/0-6`, `:10/7-9` | 70% / 30% |

---

## 8.3 Best Failover + PCC + ECMP Combination

### The Production Formula

```
PCC (traffic distribution)
  + Failover (gateway monitoring)
  + Per-WAN NAT (address translation)
  + Stateful Firewall (security)
  + Netwatch (advanced monitoring)
  = Production-Ready Multi-WAN
```

### What Each Component Handles

| Component | Responsibility | When It Activates |
|-----------|---------------|-------------------|
| PCC | Distribute new connections across WANs | Every new connection |
| Failover | Switch to backup when gateway dies | Gateway ping failure |
| Per-WAN NAT | Correct public IP per egress path | Every translated packet |
| Firewall | Block unauthorized traffic | Every packet |
| Netwatch | Disable PCC route for dead WAN | Gateway monitor failure |
| ECMP | **Not used** in NAT environments | N/A |

### ECMP Role in This Architecture

ECMP is **not part of the production combination** when NAT is involved. ECMP remains available as:

- Alternative for routed public IP deployments
- Datacenter BGP scenarios
- Internal router-to-router paths

---

## 8.4 Performance Recommendations

| Area | Recommendation | Impact |
|------|---------------|--------|
| Disable FastTrack | Prevents mangle bypass | Critical for PCC |
| Router DNS cache | Reduces WAN DNS traffic | Medium |
| TCP timeout tuning | Prevents connection table bloat | High at scale |
| Hardware router (CCR) | Handles mangle at line rate | Critical > 200 users |
| Minimal mangle rules | 6 rules for 3-WAN — no more | CPU savings |
| Netwatch interval 10s | Balance detection speed vs false positives | Stability |
| Config export schedule | Weekly automated backup | Recovery |

---

## 8.5 Stability Recommendations

| Area | Recommendation | Impact |
|------|---------------|--------|
| check-gateway on ALL routes | No blackholed traffic | Critical |
| Anti-loop WAN-to-WAN drop | Prevents routing loops | Critical |
| NTP enabled | Accurate troubleshooting logs | High |
| Disable unused services | Reduces attack surface | High |
| rp-filter=strict | Prevents IP spoofing | Medium |
| Monitor external IP (8.8.8.8) | Detect ISP-side failures | High |
| Document IP plan | Faster troubleshooting | High |
| Test failover monthly | Verify backup path works | Critical |

---

## 8.6 Deployment Checklist

| # | Task | Status |
|---|------|--------|
| 1 | Interface lists (WAN/LAN) configured | ☐ |
| 2 | IP addresses assigned per WAN and LAN | ☐ |
| 3 | Recursive gateway routes created | ☐ |
| 4 | Routing tables created (to-WAN1/2/3) | ☐ |
| 5 | PCC routes in per-WAN tables with check-gateway | ☐ |
| 6 | Failover routes in main table with distance priority | ☐ |
| 7 | PCC mangle rules with passthrough=yes | ☐ |
| 8 | Routing mark mangle rules configured | ☐ |
| 9 | Per-WAN masquerade NAT rules | ☐ |
| 10 | Stateful firewall rules (established first) | ☐ |
| 11 | Anti-loop and WAN input drop rules | ☐ |
| 12 | Netwatch scripts for gateway monitoring | ☐ |
| 13 | DNS and DHCP configured | ☐ |
| 14 | Unused services disabled | ☐ |
| 15 | NTP enabled | ☐ |
| 16 | Config exported and backed up | ☐ |
| 17 | Failover tested (disconnect one WAN) | ☐ |
| 18 | Load distribution verified (mangle stats) | ☐ |
| 19 | NAT verified (connection table check) | ☐ |
| 20 | Performance baseline recorded | ☐ |

---

## 8.7 Final Engineering Conclusion

> **For MikroTik Multi-WAN production deployments with NAT, the definitive architecture is:**
>
> **PCC** for per-connection load distribution across 2–3 ISP links, with **check-gateway failover** on every route, **per-WAN masquerade NAT** for session symmetry, **stateful firewall** for security, and **Netwatch** for proactive gateway monitoring.
>
> This combination delivers session-stable load balancing, automatic failover, correct NAT behavior, and production-grade security — the same architecture deployed by ISPs and enterprises worldwide on MikroTik RouterOS.

---

## 8.8 Continue Learning — Advanced Chapters

| Topic | Chapter |
|-------|---------|
| Hairpin NAT, DNS, Port Forwarding | [Chapter 9](../09-advanced-nat-dns/README.md) |
| QoS, Bandwidth Management | [Chapter 10](../10-qos-traffic/README.md) |
| VPN over Multi-WAN | [Chapter 11](../11-vpn-multiwan/README.md) |
| IPv6 Dual-Stack | [Chapter 12](../12-ipv6-multiwan/README.md) |
| BGP Multi-Homing | [Chapter 13](../13-bgp-multihoming/README.md) |
| Monitoring & NOC | [Chapter 14](../14-monitoring-operations/README.md) |
| PPPoE, LTE, DHCP WAN | [Chapter 15](../15-wan-types/README.md) |
| Security Hardening | [Chapter 16](../16-security-hardening/README.md) |
| Real Case Studies | [Chapter 17](../17-case-studies/README.md) |
| Quick Reference | [Cheat Sheet](../appendix/cheat-sheet.md) |

## 8.9 Reference Chapters (18–24)

| Topic | Chapter |
|-------|---------|
| RouterOS License Levels | [Chapter 18](../18-routeros-licensing/README.md) |
| Hardware Sizing | [Chapter 19](../19-hardware-selection/README.md) |
| ROS 6 → 7 Migration | [Chapter 20](../20-migration-upgrade/README.md) |
| VRRP / HA | [Chapter 21](../21-high-availability/README.md) |
| Hotspot + PCC | [Chapter 22](../22-hotspot-captive-portal/README.md) |
| Automation Scripts | [Chapter 23](../23-automation-scripting/README.md) |
| Disaster Recovery | [Chapter 24](../24-disaster-recovery/README.md) |
| FAQ | [Appendix C](../appendix/faq.md) |
| Common Mistakes | [Appendix D](../appendix/common-mistakes.md) |
| Security Labs & CVEs | [Chapter 25](../25-security-labs-cve/README.md) |

---

**[← Back to Main Index](../../../README.md)** | **[License →](../../../LICENSE)**


