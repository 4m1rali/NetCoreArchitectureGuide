# Chapter 6 — Debugging & Troubleshooting

> Real-world failure analysis and MikroTik diagnostic tools.

---

## 6.1 Why Internet Keeps Disconnecting

### Root Causes

| Cause | Symptom | Diagnosis |
|-------|---------|-----------|
| Gateway flapping | Intermittent 5–30s outages | `/ip route print detail` — routes toggling active/inactive |
| check-gateway false positive | Outages during congestion | Ping timeout on loaded link |
| DNS failure | "Connected" but pages don't load | `/ping 8.8.8.8` works, `/ping google.com` fails |
| Connection table full | Gradual degradation | `/ip firewall connection print count-only` |
| ISP-side issue | Gateway reachable, no internet | Ping gateway OK, ping 8.8.8.8 fails |
| NAT table exhaustion | New connections fail | `/ip firewall nat print` — high entry count |

### Diagnostic Commands

```
/ip route print detail where inactive
/ip route print detail where check-gateway
/ping 203.0.113.1 count=50
/ping 8.8.8.8 count=50
/tool netwatch print
/log print where topics~"route"
```

### Resolution Steps

1. Check route status: `/ip route print detail`
2. Verify gateway reachability: `/ping <gateway-ip> count=20`
3. Verify internet reachability: `/ping 8.8.8.8 count=20`
4. Check for route flapping in logs: `/log print`
5. Increase check-gateway timeout if false positives
6. Add Netwatch for upstream monitoring (8.8.8.8)

---

## 6.2 Why Load Balancing Becomes Unstable

### Root Causes

| Cause | Symptom | Diagnosis |
|-------|---------|-----------|
| Missing passthrough=yes | Random WAN per packet | `/ip firewall mangle print` — check passthrough |
| Mangle rule order wrong | Uneven distribution | `/ip firewall mangle print stats` |
| FastTrack bypassing mangle | Some connections unmarked | `/ip firewall filter print` — fasttrack rules |
| Connection mark not persisting | Sessions switch WAN mid-stream | `/ip firewall connection print` — check marks |
| One WAN route inactive | 50% traffic instead of 33% | `/ip route print` — inactive routes |
| Asymmetric routing | Downloads work, uploads fail | `/tool torch` — check both directions |

### Diagnostic Commands

```
/ip firewall mangle print stats
/ip firewall connection print where connection-mark!=""
/ip route print where routing-table~"to-WAN"
/tool torch interface=ether5 duration=30
/interface monitor-traffic ether1,ether2,ether3 once
```

### Distribution Verification

```
/ip firewall connection print count-only where connection-mark="WAN1-conn"
/ip firewall connection print count-only where connection-mark="WAN2-conn"
/ip firewall connection print count-only where connection-mark="WAN3-conn"
```

Expected: roughly equal counts across all three WAN marks.

### Resolution Steps

1. Verify mangle rule order and passthrough setting
2. Disable FastTrack if active (conflicts with PCC mangle)
3. Confirm all PCC routing table routes are active
4. Check connection mark distribution (should be ~33/33/33)
5. Monitor per-interface traffic for balance confirmation

---

## 6.3 Why Wrong Routing Occurs

### Root Causes

| Cause | Symptom | Diagnosis |
|-------|---------|-----------|
| Missing routing table | Traffic uses main table | `/routing table print` |
| Routing mark not set | PCC ignored | `/ip firewall mangle print stats` |
| Recursive route missing | Gateway unreachable | `/ip route print detail where dst-address~"GW-IP"` |
| Scope/target-scope wrong | Route not used for forwarding | `/ip route print detail` — check scope |
| Distance conflict | Failover route overrides PCC | Compare main vs PCC table routes |
| VRF misconfiguration | Traffic in wrong FIB | `/routing table print` |

### Diagnostic Commands

```
/ip route print detail
/routing table print
/ip route print where routing-mark!=""
/ip firewall mangle print stats
/routing rule print
```

### Trace Route Decision

```
/tool sniffer quick interface=ether5 ip-protocol=tcp port=443
/tool traceroute 8.8.8.8
```

### Resolution Steps

1. Verify routing tables exist: `/routing table print`
2. Confirm PCC routes are in correct tables (not main)
3. Check recursive gateway routes have scope=10
4. Verify mangle routing-mark rules have traffic
5. Ensure no conflicting routes in main table with lower distance

---

## 6.4 Why NAT Conflicts Occur

### Root Causes

| Cause | Symptom | Diagnosis |
|-------|---------|-----------|
| Single masquerade for all WANs | Return traffic via wrong WAN | `/ip firewall nat print` |
| NAT before routing mark | Wrong out-interface selected | Rule order in nat chain |
| Duplicate NAT rules | Double translation | `/ip firewall nat print` — duplicate entries |
| Port collision | Random connection failures | Two WANs NAT same port simultaneously |
| Hairpin NAT missing | Internal access to public IP fails | No dstnat loopback rule |
| Connection mark mismatch | NAT applied but wrong WAN IP | Cross-reference conn mark and NAT interface |

### Diagnostic Commands

```
/ip firewall nat print
/ip firewall connection print where src-address~"192.168"
/tool sniffer quick interface=ether2 ip-address=198.51.100.2
/ip firewall nat print stats
```

### Resolution Steps

1. Ensure separate masquerade rules per out-interface
2. Verify NAT out-interface matches PCC routing path
3. Check for duplicate or conflicting NAT rules
4. Confirm connection tracking shows correct translated addresses
5. Add hairpin NAT if internal servers use public DNS

---

## 6.5 MikroTik Debug Toolkit

### Essential Debug Commands

| Tool | Command | Purpose |
|------|---------|---------|
| Route status | `/ip route print detail` | Active/inactive routes, gateways |
| Connection table | `/ip firewall connection print` | Active sessions, marks, NAT |
| Mangle stats | `/ip firewall mangle print stats` | Rule hit counters |
| NAT stats | `/ip firewall nat print stats` | NAT rule hit counters |
| Traffic monitor | `/interface monitor-traffic <iface>` | Real-time bandwidth per WAN |
| Torch | `/tool torch <iface>` | Live connection analyzer |
| Sniffer | `/tool sniffer quick <iface>` | Packet capture |
| Traceroute | `/tool traceroute <ip>` | Path verification |
| Ping | `/ping <ip> count=50` | Latency and loss |
| Bandwidth test | `/tool bandwidth-test` | WAN throughput measurement |
| Netwatch | `/tool netwatch print` | Gateway monitor status |
| Logs | `/log print` | System events and errors |

### Debug Workflow

```
STEP 1: Verify physical layer
  /interface print stats
  /interface ethernet print

STEP 2: Verify IP and gateway
  /ip address print
  /ip route print detail

STEP 3: Verify PCC classification
  /ip firewall mangle print stats
  /ip firewall connection print count-only

STEP 4: Verify NAT
  /ip firewall nat print stats
  /ip firewall connection print where protocol=tcp

STEP 5: Verify traffic flow
  /interface monitor-traffic ether1,ether2,ether3
  /tool torch interface=ether5

STEP 6: Check logs
  /log print where topics~"firewall,route,system"
```

### Packet Sniffer for PCC Debug

```
/tool sniffer
set filter-interface=ether5 filter-ip-protocol=tcp \
    filter-connection-mark=WAN2-conn streaming-enabled=yes

/tool sniffer quick interface=ether5 duration=10
```

---

## 6.6 Common Problem → Solution Quick Reference

| Problem | Quick Fix |
|---------|-----------|
| No internet at all | Check default route active, ping gateway |
| Internet via one WAN only | PCC mangle rules missing or disabled |
| Intermittent disconnects | check-gateway too aggressive, increase timeout |
| Slow browsing | DNS issue — check `/ip dns print` |
| VPN doesn't work | Policy route VPN to single WAN |
| VoIP choppy | Policy route VoIP to lowest-latency WAN |
| Upload works, download doesn't | NAT conflict — check per-WAN masquerade |
| High CPU | Too many mangle rules or connection table full |
| After reboot no balancing | Verify startup order, routing tables persist |
| One WAN always at 0% | Route inactive — check-gateway failing |

---

**Next Chapter →** [Chapter 7: Engineering Analysis](../07-engineering-analysis/README.md)
