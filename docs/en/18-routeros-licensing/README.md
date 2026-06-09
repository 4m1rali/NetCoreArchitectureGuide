# Chapter 18 — RouterOS Licensing

> MikroTik license levels, feature requirements, and Multi-WAN compatibility.

---

## 18.1 License Level Overview

MikroTik RouterOS uses a **software license level** tied to each device. The license determines which advanced features are available — critical for Multi-WAN design.

| Level | Name | Typical Use | Multi-WAN Suitability |
|-------|------|-------------|----------------------|
| 3 | Level 3 | SOHO, basic routing | Limited — no BGP |
| 4 | Level 4 | Small business | **Minimum for production Multi-WAN** |
| 5 | Level 5 | Enterprise | Full feature set |
| 6 | Level 6 | ISP / Datacenter | Unlimited BGP, full table |

### Check Current License

```
/system license print
/system resource print
```

---

## 18.2 Feature Availability by License Level

| Feature | Level 3 | Level 4 | Level 5 | Level 6 |
|---------|---------|---------|---------|---------|
| Static routing | Yes | Yes | Yes | Yes |
| ECMP | Yes | Yes | Yes | Yes |
| PCC / Mangle | Yes | Yes | Yes | Yes |
| OSPF | No | Yes | Yes | Yes |
| BGP | No | Yes (limited) | Yes | Yes (unlimited) |
| MPLS | No | Yes | Yes | Yes |
| VRF | No | Yes | Yes | Yes |
| IPv6 full | Yes | Yes | Yes | Yes |
| WireGuard | Yes | Yes | Yes | Yes |
| IPsec | Yes | Yes | Yes | Yes |
| Hotspot | Yes | Yes | Yes | Yes |
| User Manager | No | Yes | Yes | Yes |
| The Dude | Yes | Yes | Yes | Yes |
| Container | No | No | Yes (ARM/x86) | Yes |

---

## 18.3 Multi-WAN Minimum Requirements

| Deployment | Min License | Min Hardware | Reason |
|------------|-------------|--------------|--------|
| SOHO 2-WAN failover | Level 3 | hEX / RB750 | Basic routing sufficient |
| Enterprise 3-WAN PCC | **Level 4** | RB4011 | VRF, OSPF if needed |
| ISP WISP 300 users | **Level 5** | CCR2004+ | BGP, MPLS, high conn count |
| Datacenter BGP | **Level 6** | CCR2116+ | Full BGP table support |
| Branch VPN + PCC | Level 4 | RB4011 | IPsec + mangle |

---

## 18.4 License and BGP

| Level | BGP Routes Limit | BGP Peers |
|-------|-----------------|-----------|
| 4 | ~1000 routes | 10 peers |
| 5 | ~4000 routes | 50 peers |
| 6 | Unlimited | Unlimited |

For Multi-WAN with BGP default route only (not full table), **Level 4 is sufficient**.

For full internet BGP table (750,000+ routes), **Level 6 + 8GB+ RAM** is mandatory.

---

## 18.5 Trial and CHR Licensing

### Cloud Hosted Router (CHR)

| CHR License | Speed Limit | Multi-WAN |
|-------------|-------------|-----------|
| Free | 1 Mbps | Lab only |
| P1 ($45) | 1 Gbps | Small deployment |
| P10 ($95) | 10 Gbps | Enterprise |
| P-Unlimited ($250) | Unlimited | ISP/Datacenter |

CHR is ideal for virtualized Multi-WAN lab testing and VMware/Hyper-V deployments.

### Trial License

New MikroTik hardware includes a **60-day Level 6 trial**. Use this period for full Multi-WAN testing before license settles to purchased level.

---

## 18.6 License Upgrade Path

```
Level 3 → Level 4: Purchase upgrade key from MikroTik distributor
Level 4 → Level 5: Same process
Level 5 → Level 6: Same process

/system license print
# Note the software-id, purchase key for that ID
/system license renew account=your-mikrotik-account password=xxx license-key=KEY
```

---

## 18.7 License Impact on Design Decisions

| If License Is... | Design Recommendation |
|-----------------|----------------------|
| Level 3 only | Failover only — no BGP, no VRF |
| Level 4 | PCC + Failover — full production capable |
| Level 5 | Add BGP, MPLS, advanced QoS |
| Level 6 | Full ISP architecture with BGP multi-homing |

---

**Next Chapter →** [Chapter 19: Hardware Selection](../19-hardware-selection/README.md)
