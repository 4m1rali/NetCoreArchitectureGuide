# Chapter 7 — Engineering Analysis (Expert View)

> Strategic analysis for production Multi-WAN decision-making.

---

## 7.1 When ECMP Is Appropriate — and When It Is Dangerous

### Appropriate Conditions

| Condition | Reason |
|-----------|--------|
| No NAT (routed public IP blocks) | No translation symmetry required |
| BGP multi-homed setup | Standard protocol-level ECMP |
| Datacenter outbound with /24+ public blocks | Each server has its own public IP |
| RouterOS 7 with `ecmp-per-connection=yes` | Improved session handling |
| Internal traffic between routers | No connection tracking dependency |
| Testing and lab environments | Simplicity over stability |

### Dangerous Conditions

| Condition | Risk |
|-----------|------|
| Masquerade NAT on all WANs | **CRITICAL** — sessions break randomly |
| VoIP / SIP traffic | Mid-call path switching causes drops |
| HTTPS with session cookies | TLS sessions reset on path change |
| VPN tunnels (IPsec/OpenVPN) | Tunnel renegotiation on every path change |
| Online gaming | NAT type becomes Strict, latency spikes |
| Financial / trading applications | Microsecond path changes cause order failures |
| Any session-sensitive application | User-visible disconnections |

### Expert Verdict

> **ECMP with NAT is the single most common cause of Multi-WAN failure in production MikroTik deployments.** Use PCC instead unless you have routed public IPs.

---

## 7.2 Why PCC Is the Real Multi-WAN Standard

### Technical Justification

| Factor | PCC Advantage |
|--------|--------------|
| Connection mark persistence | Entire session on one WAN — no mid-stream switching |
| Per-WAN routing tables | Independent path control per ISP |
| Per-WAN NAT | Correct masquerade per egress interface |
| Firewall compatibility | Stateful rules work with connection marks |
| Deterministic hashing | Reproducible, testable distribution |
| MikroTik native support | Built-in `per-connection-classifier` — no external tools |
| ISP industry adoption | Standard deployment pattern worldwide |

### Why Not Other Methods

| Method | Why Not Standard |
|--------|-----------------|
| NTH (deprecated) | Replaced by PCC in RouterOS 6.30+ |
| ECMP + NAT | Session breakage — unacceptable in production |
| Policy routing only | No automatic distribution — manual per-subnet |
| OSPF/BGP for SOHO | Overkill, requires ISP cooperation |
| Third-party bonding | Requires matching equipment at both ends |

### Expert Verdict

> **PCC is the de facto standard for MikroTik Multi-WAN because it is the only built-in method that simultaneously solves load distribution, NAT compatibility, and session stability.**

---

## 7.3 When Failover Is Essential

### Mandatory Scenarios

| Scenario | Why Failover Is Required |
|----------|--------------------------|
| Any production Multi-WAN | Without failover, dead WAN = blackholed traffic |
| ISP with SLA requirements | Downtime must be < 30 seconds |
| VoIP / telephony | Calls must survive WAN failure |
| Credit card processing | Transaction continuity required |
| Remote branch offices | No on-site technician to manually switch |
| LTE backup links | Backup only activates on primary failure |

### Failover Alone (Without Load Balancing)

Acceptable when:

- Secondary WAN is significantly slower (LTE backup)
- Cost difference between ISPs makes balancing uneconomical
- Only 2 WANs and primary has sufficient capacity
- Regulatory requirement for redundant path (not aggregated bandwidth)

### Expert Verdict

> **Failover is not optional in production — it is a mandatory companion to any load balancing method. Deploy check-gateway on every WAN route without exception.**

---

## 7.4 Best Production Combination

### Recommended Architecture

```
┌─────────────────────────────────────────────┐
│           PRODUCTION STACK                   │
│                                              │
│   ┌─────────────────────────────────────┐   │
│   │  PCC (Primary Load Distribution)    │   │
│   │  • per-connection-classifier 3/0,1,2│   │
│   │  • Separate routing tables          │   │
│   │  • Per-WAN masquerade NAT           │   │
│   └─────────────────────────────────────┘   │
│                      +                       │
│   ┌─────────────────────────────────────┐   │
│   │  Failover (Gateway Monitoring)      │   │
│   │  • check-gateway=ping on all routes │   │
│   │  • Netwatch scripts for PCC disable │   │
│   │  • Main table backup routes         │   │
│   └─────────────────────────────────────┘   │
│                      +                       │
│   ┌─────────────────────────────────────┐   │
│   │  Stateful Firewall                  │   │
│   │  • established/related first        │   │
│   │  • Anti-loop WAN-to-WAN drop        │   │
│   │  • WAN input drop                   │   │
│   └─────────────────────────────────────┘   │
│                      +                       │
│   ┌─────────────────────────────────────┐   │
│   │  Policy Routing (Optional)        │   │
│   │  • VoIP → lowest latency WAN      │   │
│   │  • Servers → specific WAN         │   │
│   │  • Management → dedicated WAN     │   │
│   └─────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

### What NOT to Combine

| Combination | Problem |
|-------------|---------|
| ECMP + PCC on same traffic | Conflicting routing decisions |
| PCC + ECMP on same traffic | Double classification |
| Failover without check-gateway | Dead routes stay active |
| PCC without per-WAN NAT | NAT conflict on return path |
| FastTrack + PCC | FastTrack bypasses mangle marks |

---

## 7.5 Performance and Stability Recommendations

### Hardware Sizing

| Users | Router Model | RAM | Expected PCC CPU |
|-------|-------------|-----|-----------------|
| 1–50 | RB750Gr3 | 256MB | < 5% |
| 50–200 | RB4011 | 1GB | 5–15% |
| 200–1000 | CCR2004 | 4GB | 10–25% |
| 1000+ | CCR2116/CCR2216 | 16GB | 15–30% |

### Optimization Guidelines

| Area | Recommendation |
|------|---------------|
| Mangle rules | Minimum necessary — 6 rules for 3-WAN PCC |
| Connection tracking | Set appropriate TCP timeouts |
| FastTrack | Disable if using PCC mangle |
| DNS | Use router DNS cache — reduce WAN DNS queries |
| Monitoring | Netwatch every 10s, not every 1s |
| Logging | Disable firewall debug logging in production |
| NTP | Always enable — accurate logs for troubleshooting |
| Backup | Export config before every change |

### Connection Tracking Tuning

```
/ip firewall connection tracking
set enabled=yes tcp-established-timeout=1d tcp-time-wait-timeout=10s udp-timeout=30s
```

---

## 7.6 ISP vs Enterprise vs Datacenter

| Aspect | ISP | Enterprise | Datacenter |
|--------|-----|------------|------------|
| Primary method | PCC + BGP | PCC + Failover | BGP + ECMP |
| WAN count | 3–10+ | 2–3 | 4+ |
| NAT | Per-subscriber | Per-WAN masquerade | None (public IPs) |
| Failover | Netwatch + BGP | check-gateway | BGP failover |
| Monitoring | SNMP + Netwatch | Netwatch + Syslog | BGP + SNMP |
| Complexity | Very High | High | Medium (with BGP) |
| Session sensitivity | High (subscribers) | High (users) | Low (stateless) |

---

**Next Chapter →** [Chapter 8: Final Architecture Summary](../08-final-summary/README.md)
