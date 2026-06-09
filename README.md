# MikroTik Multi-WAN Enterprise Architecture Guide

> **Senior ISP Network Architect · MikroTik Expert · Enterprise Network Designer**
>
> Production-ready documentation for ISP / Enterprise / Datacenter Multi-WAN deployments on MikroTik RouterOS.

---

## Language Selection / Выбор языка / انتخاب زبان

| Language | Code | Documentation Path |
|----------|------|--------------------|
| **English** | `EN` | [docs/en/](docs/en/) |
| **فارسی (Persian)** | `FA` | [docs/fa/](docs/fa/) |
| **Русский (Russian)** | `RU` | [docs/ru/](docs/ru/) |

| License | [LICENSE](LICENSE) · [فارسی](LICENSE.fa.md) · [Русский](LICENSE.ru.md) |

---

## Executive Summary

This guide is a complete, folder-by-folder engineering book for designing, deploying, and operating **Multi-WAN networks on MikroTik RouterOS** at production scale.

After completing this documentation, you will be able to:

- Design and deploy a real **Multi-WAN** topology with 2–3 ISPs
- Implement **Load Balancing**, **Failover**, **ECMP**, and **PCC** at production level
- Manage **NAT**, **Routing**, **Firewall**, **QoS**, and **VPN** as an integrated system
- Configure **PPPoE**, **LTE**, **DHCP**, and **BGP** WAN handoffs
- Deploy **IPv6** dual-stack Multi-WAN
- Debug instability, routing errors, and NAT conflicts in live networks
- Monitor and operate Multi-WAN with SNMP, Syslog, and Netwatch
- Apply real-world case studies from ISP, Enterprise, and Datacenter deployments

---

## Book Structure (Folder-by-Folder)

```
NetCoreArchitectureGuide/
├── README.md
└── docs/
    ├── en/  fa/  ru/
    │
    ├── 01-network-architecture/     ← Foundation: NAT, Routing, Conntrack, VRF, MTU
    ├── 02-core-concepts/            ← ECMP, PCC, Failover, Policy Routing, Recursive
    ├── 03-comparison-table/         ← Enterprise method comparison matrix
    ├── 04-production-design/        ← 3-ISP real scenario + traffic flow
    ├── 05-mikrotik-configuration/   ← Full production RouterOS scripts
    ├── 06-debugging-troubleshooting/← Real-world failure analysis
    ├── 07-engineering-analysis/     ← Expert decision framework
    ├── 08-final-summary/            ← Best practices + deployment checklist
    │
    ├── 09-advanced-nat-dns/         ← Hairpin NAT, DNS, inbound LB, port forwarding
    ├── 10-qos-traffic/              ← Bandwidth management, PCQ, priority queues
    ├── 11-vpn-multiwan/             ← IPsec, WireGuard, L2TP over Multi-WAN
    ├── 12-ipv6-multiwan/            ← IPv6 PCC, NPT, DHCPv6
    ├── 13-bgp-multihoming/          ← BGP peering, ASN, PI space
    ├── 14-monitoring-operations/    ← SNMP, Syslog, alerts, runbook
    ├── 15-wan-types/                ← PPPoE, LTE, DHCP, VLAN handoffs
    ├── 16-security-hardening/       ← DDoS, anti-spoof, RAW chain
    ├── 17-case-studies/             ← 5 real-world deployment stories
    ├── 18-routeros-licensing/       ← License levels, CHR, feature matrix
    ├── 19-hardware-selection/       ← CCR/RB sizing, CPU/RAM planning
    ├── 20-migration-upgrade/        ← RouterOS 6 → 7 migration guide
    ├── 21-high-availability/        ← VRRP, dual router, BGP HA
    ├── 22-hotspot-captive-portal/   ← Guest WiFi + PCC
    ├── 23-automation-scripting/     ← Scripts, schedulers, REST API
    ├── 24-disaster-recovery/        ← Backup, recovery, RTO/RPO
    ├── 25-security-labs-cve/        ← Labs, CVEs, overlooked security
    └── appendix/
        ├── glossary.md
        ├── cheat-sheet.md
        ├── faq.md
        ├── common-mistakes.md
        └── wireless-lte-guide.md
```

---

## Chapter Index (English)

### Part I — Foundations

| # | Chapter | Description | Link |
|---|---------|-------------|------|
| 1 | **Network Architecture** | Multi-WAN, NAT, routing tables, conntrack, L2/L3/L4, VRF, MTU, FastTrack | [→](docs/en/01-network-architecture/README.md) |
| 2 | **Core Concepts** | ECMP, PCC, Failover, Load Balancing, Policy Routing, Recursive Routing | [→](docs/en/02-core-concepts/README.md) |
| 3 | **Comparison Table** | Enterprise method comparison matrix | [→](docs/en/03-comparison-table/README.md) |

### Part II — Design & Configuration

| # | Chapter | Description | Link |
|---|---------|-------------|------|
| 4 | **Production Design** | 3-ISP scenario, topology, traffic flow, NAT journey | [→](docs/en/04-production-design/README.md) |
| 5 | **MikroTik Configuration** | Full RouterOS: WAN, PCC, Failover, NAT, QoS, PPPoE, Hairpin | [→](docs/en/05-mikrotik-configuration/README.md) |

### Part III — Operations

| # | Chapter | Description | Link |
|---|---------|-------------|------|
| 6 | **Debugging** | Internet drops, LB instability, routing errors, NAT conflicts | [→](docs/en/06-debugging-troubleshooting/README.md) |
| 7 | **Engineering Analysis** | When to use ECMP, PCC, Failover — expert view | [→](docs/en/07-engineering-analysis/README.md) |
| 8 | **Final Summary** | Best-practice design + 20-point deployment checklist | [→](docs/en/08-final-summary/README.md) |

### Part IV — Advanced Topics

| # | Chapter | Description | Link |
|---|---------|-------------|------|
| 9 | **Advanced NAT & DNS** | Hairpin NAT, DNS strategies, inbound LB, port forwarding | [→](docs/en/09-advanced-nat-dns/README.md) |
| 10 | **QoS & Traffic** | Queue trees, PCQ, VoIP priority, per-WAN shaping | [→](docs/en/10-qos-traffic/README.md) |
| 11 | **VPN over Multi-WAN** | IPsec, WireGuard, VPN failover | [→](docs/en/11-vpn-multiwan/README.md) |
| 12 | **IPv6 Multi-WAN** | IPv6 PCC, NPT, DHCPv6, dual-stack | [→](docs/en/12-ipv6-multiwan/README.md) |
| 13 | **BGP Multi-Homing** | BGP peering, ASN, path selection | [→](docs/en/13-bgp-multihoming/README.md) |
| 14 | **Monitoring** | SNMP, Syslog, Netwatch, email alerts, runbook | [→](docs/en/14-monitoring-operations/README.md) |
| 15 | **WAN Types** | PPPoE, LTE, DHCP, VLAN, mixed WAN | [→](docs/en/15-wan-types/README.md) |
| 16 | **Security** | DDoS, anti-spoof, RAW chain, geo-block | [→](docs/en/16-security-hardening/README.md) |
| 17 | **Case Studies** | ISP WISP, Enterprise HQ, Datacenter, Branch, Hotel | [→](docs/en/17-case-studies/README.md) |

### Part V — Operations & Reference

| # | Chapter | Description | Link |
|---|---------|-------------|------|
| 18 | **RouterOS Licensing** | License levels, CHR, feature matrix, upgrade path | [→](docs/en/18-routeros-licensing/README.md) |
| 19 | **Hardware Selection** | CCR/RB sizing, CPU/RAM, interface planning | [→](docs/en/19-hardware-selection/README.md) |
| 20 | **Migration & Upgrade** | RouterOS 6 → 7 safe migration procedure | [→](docs/en/20-migration-upgrade/README.md) |
| 21 | **High Availability** | VRRP, dual router, BGP-based HA | [→](docs/en/21-high-availability/README.md) |
| 22 | **Hotspot** | Captive portal + PCC for guest WiFi | [→](docs/en/22-hotspot-captive-portal/README.md) |
| 23 | **Automation** | Scripts, schedulers, REST API, alerts | [→](docs/en/23-automation-scripting/README.md) |
| 24 | **Disaster Recovery** | Backup strategy, recovery procedures, RTO/RPO | [→](docs/en/24-disaster-recovery/README.md) |
| 25 | **Security Labs & CVEs** | Hands-on labs, CVE database, overlooked security threats | [→](docs/en/25-security-labs-cve/README.md) |

### Appendix

| # | Document | Link |
|---|----------|------|
| A | Glossary | [→](docs/en/appendix/glossary.md) |
| B | Cheat Sheet | [→](docs/en/appendix/cheat-sheet.md) |
| C | FAQ | [→](docs/en/appendix/faq.md) |
| D | Top 30 Common Mistakes | [→](docs/en/appendix/common-mistakes.md) |
| E | Wireless & LTE Guide | [→](docs/en/appendix/wireless-lte-guide.md) |

---

## Quick Start Path

1. [Chapter 1](docs/en/01-network-architecture/README.md) — Understand packet flow
2. [Chapter 2](docs/en/02-core-concepts/README.md) — Learn PCC, ECMP, Failover
3. [Chapter 3](docs/en/03-comparison-table/README.md) — Choose your method
4. [Chapter 4](docs/en/04-production-design/README.md) — Study reference topology
5. [Chapter 5](docs/en/05-mikrotik-configuration/README.md) — Deploy configuration
6. [Chapter 6](docs/en/06-debugging-troubleshooting/README.md) — Debug during go-live
7. [Chapters 9–17](docs/en/09-advanced-nat-dns/README.md) — Advanced topics as needed
8. [Cheat Sheet](docs/en/appendix/cheat-sheet.md) — Quick reference

---

## Production Recommendation (TL;DR)

| Component | Recommended Method |
|-----------|-------------------|
| Load Distribution | **PCC** (Per Connection Classifier) |
| Gateway Redundancy | **Recursive Routing + check-gateway** |
| Backup Path | **Failover** with Netwatch |
| NAT | **Per-WAN masquerade** with connection marks |
| VoIP / Critical Apps | **Policy Routing** above PCC |
| QoS | **Queue tree** with packet marks |
| VPN | **One tunnel per WAN** with policy route |
| Monitoring | **SNMP + Netwatch + Syslog** |
| Security | **Stateful firewall + RAW chain + anti-spoof** |

> **Best production combination:** PCC + Policy Routing + Failover + Per-WAN NAT + QoS + Netwatch

---

## RouterOS Compatibility

| Item | Requirement |
|------|-------------|
| RouterOS Version | 7.x recommended (6.x compatible with notes) |
| Hardware | RB4011, CCR series, or equivalent |
| License | Level 4+ for advanced routing |
| Interfaces | Minimum 2 WAN + 1 LAN (3 WAN recommended) |

---

## License

This documentation is released under the **[MIT License](LICENSE)**.

You are free to use, copy, modify, merge, publish, and distribute this documentation for personal, educational, and commercial purposes.

**Disclaimer:** This is an independent educational resource. MikroTik, RouterOS, and related trademarks belong to [MikroTikls SIA](https://mikrotik.com). This project is not affiliated with or endorsed by MikroTik. Always test configurations in a lab before production deployment.

| Item | Detail |
|------|--------|
| Documentation License | MIT License |
| Copyright | 2026 NetCoreArchitectureGuide Contributors |
| RouterOS License | Separate — purchased from MikroTik per device ([Chapter 18](docs/en/18-routeros-licensing/README.md)) |

---

**[Begin Reading — Chapter 1 →](docs/en/01-network-architecture/README.md)** | **[Cheat Sheet →](docs/en/appendix/cheat-sheet.md)** | **[License →](LICENSE)**
