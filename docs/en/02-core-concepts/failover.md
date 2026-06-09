# Failover

> Automatic WAN path recovery and gateway redundancy.

---

## Engineering Definition

**Failover** is a network resilience mechanism that monitors the health of WAN gateways and automatically redirects traffic to a backup path when the primary gateway becomes unreachable. In MikroTik, failover is implemented through **gateway monitoring (check-gateway)** combined with **recursive routing** and **route distance prioritization**.

Failover operates at **Layer 3** and is independent of load balancing — it ensures continuity, not distribution.

---

## Internal Router Flow (Step-by-Step)

```
1. Static route configured with check-gateway=ping
   → Route: 0.0.0.0/0 via 203.0.113.1 distance=1 check-gateway=ping
2. Router sends ICMP ping to 203.0.113.1 every check-interval
3. Gateway responds → route status: ACTIVE
4. Traffic flows through ISP-1 normally
5. ISP-1 link fails (cable cut, ISP outage, gateway down)
6. Ping timeout → route status: INACTIVE
7. Router selects next best route:
   → 0.0.0.0/0 via 198.51.100.1 distance=2 check-gateway=ping
8. NEW connections use ISP-2
9. EXISTING connections on ISP-1: DROPPED (expected behavior)
10. ISP-1 recovers → ping success → route ACTIVE again
11. New connections can use ISP-1 (if distance=1 is preferred)
```

---

## Behavior in MikroTik RouterOS

### Basic Failover Configuration

```
/ip route
add dst-address=0.0.0.0/0 gateway=203.0.113.1 distance=1 check-gateway=ping
add dst-address=0.0.0.0/0 gateway=198.51.100.1 distance=2 check-gateway=ping
add dst-address=0.0.0.0/0 gateway=192.0.2.1 distance=3 check-gateway=ping
```

### check-gateway Options

| Option | Behavior |
|--------|----------|
| `ping` | ICMP ping to gateway IP — most common |
| `arp` | ARP resolution check — for directly connected gateways |
| `none` | No monitoring — route always active (not recommended) |

### Recursive Routing for Failover

When the gateway is not directly reachable (e.g., PPPoE or metro Ethernet):

```
/ip route
add dst-address=203.0.113.1/32 gateway=ether1 scope=10
add dst-address=0.0.0.0/0 gateway=203.0.113.1 distance=1 check-gateway=ping
```

**Flow:**
1. Route to 203.0.113.1 resolved via ether1 (scope=10)
2. Default route uses 203.0.113.1 as gateway
3. check-gateway pings 203.0.113.1
4. If unreachable, both routes become inactive

### Failover with PCC

In production, failover works **alongside** PCC:

- PCC routes in `to-WAN1` table have check-gateway on ISP-1 gateway
- When ISP-1 fails, `to-WAN1` table route becomes inactive
- PCC-classified traffic for WAN1 has no active route → **dropped**
- **Solution:** Add fallback routes or use Netwatch scripts to remove PCC marks for failed WAN

### Netwatch Advanced Failover

```
/tool netwatch
add host=203.0.113.1 interval=10s timeout=3s \
    up-script="/ip route enable [find gateway=203.0.113.1]" \
    down-script="/ip route disable [find gateway=203.0.113.1]"
```

---

## Use Cases

| Environment | Application |
|-------------|-------------|
| Enterprise HQ | Primary fiber + backup LTE failover |
| ISP Edge | Upstream provider redundancy |
| Branch Office | Dual ISP with automatic switchover |
| Critical Services | VoIP gateway must never lose connectivity |
| Any Multi-WAN | Mandatory companion to PCC or ECMP |

---

## Pros

| Advantage | Detail |
|-----------|--------|
| Automatic recovery | No manual intervention required |
| Simple configuration | Distance + check-gateway is sufficient |
| Fast detection | Ping timeout typically 3–10 seconds |
| Works with any method | Compatible with PCC, ECMP, or standalone |
| Proven reliability | Standard approach across all router vendors |

---

## Cons and Risks

| Risk | Detail |
|------|--------|
| Active session loss | Existing connections on failed WAN are dropped |
| Flapping | Unstable link causes repeated failover/failback |
| DNS cache issues | Clients may cache failed WAN's DNS responses |
| PCC orphan connections | Connections marked for dead WAN have no route |
| Asymmetric routing on recovery | Return traffic may arrive before route is restored |
| Single point of monitoring | Ping to gateway doesn't detect ISP-side failures |

### Detecting ISP-Side Failures

Gateway ping only confirms L2/L3 reachability to the ISP gateway — not internet connectivity beyond.

**Solution:** Monitor an external host:

```
/tool netwatch
add host=8.8.8.8 interval=10s timeout=3s \
    up-script="..." down-script="..."
```

Or use recursive routing with multiple monitor targets.

---

## Common Implementation Errors

| Error | Consequence | Fix |
|-------|-------------|-----|
| No check-gateway | Dead routes stay active forever | Always enable check-gateway=ping |
| Same distance on all routes | Unpredictable failover order | Use distance=1,2,3 for priority |
| Missing recursive route | Gateway unreachable, route never activates | Add host route to gateway via interface |
| Failover without PCC fallback | PCC traffic blackholed on WAN failure | Implement Netwatch scripts or disable PCC marks |
| Too aggressive ping interval | False positives on congested links | Use interval=10s timeout=3s minimum |
| No upstream monitoring | Gateway up but ISP routing broken | Monitor external IP (8.8.8.8, 1.1.1.1) |

---

**Next →** [Load Balancing](load-balancing.md)
