# Lab Exercises — Multi-WAN Practice

> 12 isolated lab exercises. Use CHR or spare hardware. Never on production.

---

## Lab Environment Setup

```
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│  Virtual ISP1│   │  Virtual ISP2│   │  Virtual ISP3│
│  203.0.113.0 │   │ 198.51.100.0 │   │  192.0.2.0   │
└──────┬───────┘   └──────┬───────┘   └──────┬───────┘
       │ ether1           │ ether2           │ ether3
       └──────────┬───────┴──────────────────┘
                  │
           ┌──────▼──────┐
           │  CHR Router │
           │  RouterOS 7 │
           └──────┬──────┘
                  │ ether4
           ┌──────▼──────┐
           │  Test LAN   │
           │ 192.168.1.0 │
           └─────────────┘
```

### Base Configuration (All Labs)

```
/interface list
add name=WAN
add name=LAN
/interface list member
add interface=ether1 list=WAN
add interface=ether2 list=WAN
add interface=ether3 list=WAN
add interface=ether4 list=LAN

/ip address
add address=203.0.113.2/30 interface=ether1
add address=198.51.100.2/30 interface=ether2
add address=192.0.2.2/30 interface=ether3
add address=192.168.1.1/24 interface=ether4
```

---

## Lab 1 — Basic PCC 3-WAN

**Objective:** Configure PCC and verify ~33/33/33 distribution.

**Tasks:**
1. Create routing tables `to-WAN1`, `to-WAN2`, `to-WAN3`
2. Add PCC routes with check-gateway
3. Configure PCC mangle rules (3/0, 3/1, 3/2)
4. Add per-WAN masquerade NAT
5. Generate traffic from 3 LAN clients simultaneously

**Verification:**
```
/ip firewall mangle print stats
/ip firewall connection print count-only where connection-mark="WAN1-conn"
/ip firewall connection print count-only where connection-mark="WAN2-conn"
/ip firewall connection print count-only where connection-mark="WAN3-conn"
/interface monitor-traffic ether1,ether2,ether3 once
```

**Success Criteria:** Each WAN carries 25–40% of new connections.

---

## Lab 2 — Failover Simulation

**Objective:** Verify automatic failover when one WAN dies.

**Tasks:**
1. Complete Lab 1 configuration
2. Generate continuous ping from LAN: `/ping 8.8.8.8 count=1000`
3. Disable ether1: `/interface disable ether1`
4. Measure downtime
5. Re-enable ether1 and observe recovery

**Verification:**
```
/log print where message~"route"
/tool netwatch print
```

**Success Criteria:** Failover < 15 seconds. Ping loss < 20 packets.

---

## Lab 3 — NAT Symmetry Test

**Objective:** Prove per-WAN NAT maintains return path.

**Tasks:**
1. Configure PCC (Lab 1)
2. From LAN client, open 10 HTTPS sessions to different sites
3. Inspect connection table for NAT translation consistency

**Verification:**
```
/ip firewall connection print where protocol=tcp and dst-port=443
```

**Success Criteria:** Each connection shows consistent `repl-src-address` matching the correct WAN public IP.

---

## Lab 4 — Weighted PCC (70/30)

**Objective:** Configure unequal load distribution.

**Tasks:**
1. Use `:10/0` through `:10/6` for WAN1, `:10/7` through `:10/9` for WAN2
2. Disable WAN3 for this lab
3. Generate 100 new TCP connections

**Success Criteria:** WAN1 ≈ 65–75% of connections, WAN2 ≈ 25–35%.

---

## Lab 5 — Policy Routing (VoIP Priority)

**Objective:** Force VoIP subnet to WAN1, rest via PCC.

**Tasks:**
1. Create address range 192.168.1.240/28 as "VoIP subnet"
2. Add mangle rule: UDP 5060, 10000-20000 → WAN1-conn (above PCC rules)
3. Generate SIP traffic from .240/28 and HTTP from .10-.50
4. Verify VoIP always uses WAN1

**Success Criteria:** 100% of UDP 5060 connections marked WAN1-conn.

---

## Lab 6 — Security Hardening Audit

**Objective:** Find and close exposed services.

**Tasks:**
1. Run `/ip service print`
2. Run `/user print`
3. Scan router from "WAN" side (Nmap from virtual ISP network):
   - `nmap -sS -p 21,22,23,80,443,8291,8728,8729 <router-wan-ip>`
4. Apply hardening from Chapter 16
5. Re-scan — all ports filtered/closed from WAN

**Success Criteria:** Zero open management ports from WAN.

---

## Lab 7 — CVE-2018-14847 Awareness (Defensive)

**Objective:** Understand Winbox vulnerability and verify patch status.

> **CVE-2018-14847** allowed reading arbitrary files via Winbox port. Fixed in RouterOS 6.42.1+ and 6.43+.

**Tasks:**
1. Deploy OLD RouterOS 6.40.x CHR instance (isolated lab only)
2. Check version: `/system resource print`
3. Verify Winbox file read is possible on unpatched version (research only)
4. Upgrade to latest long-term: `/system package update`
5. Re-test — vulnerability patched

**Success Criteria:** Understand why **never** expose Winbox to WAN and **always** patch.

---

## Lab 8 — Netwatch Script Injection Test

**Objective:** Discover unsafe Netwatch scripts.

**Tasks:**
1. Review all Netwatch entries: `/tool netwatch print`
2. Check if scripts accept external input
3. Replace unsafe scripts with parameterized safe versions
4. Test failover still works after script hardening

**Unsafe pattern:**
```
down-script="/ip route disable [find gateway=203.0.113.1]"
```

**Safer pattern:**
```
down-script={/ip route disable [find comment="PCC ISP-1"]}
```

**Success Criteria:** No scripts with hardcoded IPs that could be manipulated.

---

## Lab 9 — IPv6 Firewall Gap

**Objective:** Prove IPv4-only firewall leaves IPv6 exposed.

**Tasks:**
1. Configure IPv6 on all interfaces
2. Apply IPv4 firewall only (no `/ipv6 firewall`)
3. Attempt access from WAN via IPv6
4. Add `/ipv6 firewall filter` rules mirroring IPv4
5. Re-test — access blocked

**Success Criteria:** IPv6 WAN input dropped after adding IPv6 firewall.

---

## Lab 10 — Connection Table Exhaustion

**Objective:** Understand resource limits.

**Tasks:**
1. Set `max-entries=10000` (lab only)
2. Generate connections with `/tool traffic-generator` or flood ping
3. Monitor: `/ip firewall connection print count-only`
4. Observe behavior at limit — new connections dropped?
5. Restore `max-entries=262144`

**Success Criteria:** Document CPU and RAM at 80% connection table capacity.

---

## Lab 11 — Backup File Secret Exposure

**Objective:** Learn that `.rsc` exports contain passwords in plaintext.

**Tasks:**
1. `/export file=test-backup`
2. Open file — search for `password=`, `secret=`, `pre-shared-key=`
3. Implement password redaction before storing backups off-router
4. Use `/system backup save` (binary, slightly better) vs export

**Success Criteria:** Team policy: never store `.rsc` exports unencrypted off-router.

---

## Lab 12 — Full Multi-WAN + Security Integration

**Objective:** Production-ready lab combining all skills.

**Tasks:**
1. PCC 3-WAN + Failover + Per-WAN NAT
2. Policy routing for management subnet
3. Full security hardening (Chapter 16 + Chapter 25)
4. SNMP monitoring configured
5. Automated daily backup script
6. Document entire config with IP plan
7. Simulate WAN1 failure during active traffic
8. Run security scan from WAN — zero open ports
9. Export final config as deliverable template

**Success Criteria:** Pass all verification commands from Chapters 5, 6, and 16.

---

## Lab Scoring Rubric

| Score | Criteria |
|-------|----------|
| **Expert** | All 12 labs completed, documented, security scan clean |
| **Advanced** | Labs 1–8 completed, failover < 10s |
| **Intermediate** | Labs 1–5 completed, PCC balanced |
| **Beginner** | Labs 1–2 completed |

---

**Next →** [CVE & Exploit Reference](cve-exploit-reference.md)
