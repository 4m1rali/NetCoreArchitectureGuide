# Appendix D — Top 30 Common Mistakes

> Avoid these errors in production Multi-WAN deployments.

---

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | ECMP + masquerade NAT | Sessions break randomly | Use PCC |
| 2 | No check-gateway on routes | Dead WAN stays active | Add `check-gateway=ping` |
| 3 | FastTrack with PCC | Mangle bypassed | Disable FastTrack |
| 4 | Missing passthrough=yes | Only first packet marked | Always set passthrough=yes |
| 5 | PCC rules below policy rules | VoIP goes through PCC | Policy rules first |
| 6 | Single global masquerade | NAT conflict on return | Per-interface masquerade |
| 7 | LTE in PCC classifier | Massive data bill | Failover only for LTE |
| 8 | No MSS clamp on PPPoE | Large packets fail | MSS clamp 1440 |
| 9 | WAN interfaces in bridge | Cannot route Multi-WAN | Router mode on WAN |
| 10 | No anti-loop rule | Routing loop on failover | Drop WAN-to-WAN forward |
| 11 | No backup before change | Unrecoverable config | Export before every change |
| 12 | Testing in production | Outage during business hours | Lab test first |
| 13 | Same distance on failover routes | Unpredictable priority | distance=1,2,3 |
| 14 | No recursive route for PPPoE | Gateway unreachable | Interface as gateway |
| 15 | DNS pointed to ISP directly | Inconsistent resolution | Force DNS via router |
| 16 | Default add-default-route=yes on DHCP | DHCP overrides your routes | add-default-route=no |
| 17 | Classifier N/M wrong | Missing WAN in distribution | N=WAN count, M=0 to N-1 |
| 18 | No Netwatch on PCC routes | PCC sends to dead WAN | Netwatch disable script |
| 19 | Firewall drops established | All sessions break | Accept established first |
| 20 | Winbox open on WAN | Security breach | Restrict to LAN/VPN |
| 21 | No NTP configured | Logs useless for debug | Enable NTP |
| 22 | Connection tracking too small | Drops at peak hours | Increase max-entries |
| 23 | VRRP without preempt | Backup stays master after recovery | preempt=yes |
| 24 | BGP without route filter | Full table kills router | Filter to default only |
| 25 | QoS before PCC mangle | Wrong classification order | PCC first, QoS second |
| 26 | VPN without policy route | Tunnel traffic PCC-balanced | Force routing-mark |
| 27 | No monitoring | Failures discovered by users | SNMP + Netwatch |
| 28 | Upgrading without backup | No rollback possible | Export + binary backup |
| 29 | Ignoring MTU differences | Mysterious partial failures | Per-interface MTU + MSS |
| 30 | Level 3 license for BGP | BGP not available | Upgrade to Level 4+ |

---

## Severity Classification

| Severity | Mistakes |
|----------|----------|
| **CRITICAL** (immediate outage) | #1, #2, #4, #6, #9, #16, #19 |
| **HIGH** (degraded service) | #3, #5, #7, #8, #14, #18, #22, #26 |
| **MEDIUM** (security/stability) | #10, #15, #20, #24, #27, #29 |
| **LOW** (operational) | #11, #12, #13, #17, #21, #23, #25, #28, #30 |

---

**[← FAQ](faq.md)** | **[Cheat Sheet →](cheat-sheet.md)**
