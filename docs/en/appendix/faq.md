# Appendix C — Frequently Asked Questions (FAQ)

---

## General

### Q: Can I combine bandwidth of 3x 100Mbps into 300Mbps for one download?

**A:** No. Load balancing distributes **across connections**, not within a single connection. One download uses one WAN (50–100Mbps). But 10 simultaneous downloads can use all 3 WANs (~300Mbps aggregate).

### Q: Which method should I use?

**A:** For NAT environments (most deployments): **PCC + Failover**. For routed public IPs without NAT: **ECMP + Failover**. For ISP with own ASN: **BGP**.

### Q: Does MikroTik support true link bonding like 802.3ad?

**A:** No native L2 bonding across ISPs. PCC provides **logical** load balancing at L3/L4. For true L2 bonding, both ends must support it (rare with ISPs).

---

## PCC

### Q: Why is my PCC distribution 70/30 instead of 50/50?

**A:** PCC hashes **new connections**. If most traffic is long-lived (streaming), distribution depends on when connections started. Short tests show skew. Monitor over hours, not minutes.

### Q: Can I weight PCC for unequal WAN speeds?

**A:** Yes. Use classifier buckets: `:10/0` through `:10/6` for 70% on WAN1, `:10/7` through `:10/9` for 30% on WAN2.

### Q: Does PCC work with IPv6?

**A:** Yes. Use `/ipv6 firewall mangle` with the same classifier syntax.

---

## Failover

### Q: How fast is failover?

**A:** Typically 3–15 seconds with `check-gateway=ping`. Netwatch scripts add 10–30 seconds. BGP failover: 30–90 seconds (hold timer).

### Q: Will active connections survive failover?

**A:** No. Connections on the failed WAN are dropped. New connections use the backup WAN. This is expected behavior.

### Q: Should LTE be in PCC?

**A:** Generally **no**. Use LTE as failover-only (distance=3) to avoid unexpected data charges.

---

## NAT

### Q: Why does NAT break with ECMP?

**A:** ECMP per-packet hashing sends packets of the same connection through different WANs. Return traffic arrives on the wrong WAN, breaking connection tracking.

### Q: What is hairpin NAT?

**A:** Allows LAN clients to access your public IP from inside the network. See [Chapter 9](../09-advanced-nat-dns/README.md).

---

## Performance

### Q: My router CPU is at 80% with PCC. What do I do?

**A:** Upgrade to CCR hardware, reduce mangle rules, disable unnecessary logging, ensure FastTrack is off (it conflicts with PCC anyway).

### Q: How many connections can a RB4011 handle with PCC?

**A:** ~200,000 active connections, ~5,000 new connections/second with PCC mangle.

---

## Licensing

### Q: What license do I need for Multi-WAN?

**A:** Level 4 minimum for production. Level 3 works for basic failover only. See [Chapter 18](../18-routeros-licensing/README.md).

### Q: Is CHR free?

**A:** CHR has a free license limited to 1 Mbps. Paid licenses ($45–$250) unlock full speed.

---

## Troubleshooting

### Q: Internet works on one WAN only. PCC not balancing?

**A:** Check: mangle stats (are rules hit?), route inactive status, `passthrough=yes` on marks, FastTrack disabled.

### Q: After RouterOS upgrade, PCC stopped working?

**A:** RouterOS 7 uses `/routing table` instead of `/routing mark`. See [Chapter 20](../20-migration-upgrade/README.md).

---

**[← Cheat Sheet](cheat-sheet.md)** | **[Glossary →](glossary.md)**
