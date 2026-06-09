# Policy Routing

> Application-aware and destination-aware WAN path selection.

---

## Engineering Definition

**Policy Routing** directs specific traffic to specific WAN paths based on criteria other than destination prefix alone — source subnet, protocol, port, user, time, or connection bytes. In MikroTik, policy routing is implemented via **mangle rules** (connection/routing marks), **routing rules**, and **address lists**.

Unlike PCC (which distributes all traffic evenly), policy routing provides **deterministic path control**.

---

## Internal Router Flow

```
1. Packet arrives from LAN
2. Mangle PREROUTING — policy rules evaluated FIRST (before PCC)
   a. Match: src-address=192.168.1.0/28 → mark WAN1-conn (VoIP subnet)
   b. Match: protocol=tcp dst-port=22 → mark WAN3-conn (management)
   c. Match: dst-address-list=streaming → mark WAN2-conn
3. If no policy match → PCC classifier runs (fallback distribution)
4. Routing mark applied → correct routing table selected
5. NAT and forwarding proceed normally
```

**Rule order is critical:** Policy rules must be placed **above** PCC classifier rules.

---

## MikroTik Configuration Patterns

### VoIP to Lowest-Latency WAN

```
/ip firewall mangle
add chain=prerouting src-address=192.168.1.240/28 protocol=udp \
    dst-port=5060,5061,10000-20000 action=mark-connection \
    new-connection-mark=WAN1-conn passthrough=yes comment="VoIP → ISP-1"
```

### Management Traffic to Dedicated WAN

```
/ip firewall mangle
add chain=prerouting src-address=192.168.1.0/28 protocol=tcp \
    dst-port=22,8291 action=mark-connection \
    new-connection-mark=WAN3-conn passthrough=yes comment="Mgmt → ISP-3"
```

### Force Specific User to Specific WAN

```
/ip firewall address-list
add list=user-john address=192.168.1.55

/ip firewall mangle
add chain=prerouting src-address-list=user-john action=mark-connection \
    new-connection-mark=WAN2-conn passthrough=yes
```

### Routing Rules (RouterOS 7)

```
/routing rule
add src-address=192.168.1.0/24 action=lookup-only-in-table table=to-WAN1
```

---

## Use Cases

| Environment | Policy |
|-------------|--------|
| Enterprise | VoIP → fiber, bulk → cheaper WAN |
| ISP | Premium customers → low-latency upstream |
| Branch office | ERP system → primary WAN only |
| Gaming lounge | Gaming subnet → lowest latency ISP |
| Compliance | Financial traffic → audited WAN path |

---

## Pros

| Advantage | Detail |
|-----------|--------|
| Granular control | Per-subnet, per-protocol, per-user |
| SLA compliance | Critical apps always on best path |
| Cost optimization | Bulk on cheap link, realtime on premium |
| Combines with PCC | Policy first, PCC distributes remainder |
| Audit trail | Address lists document traffic policy |

---

## Cons and Risks

| Risk | Detail |
|------|--------|
| Rule explosion | Too many rules → CPU overhead |
| Maintenance burden | IP changes require list updates |
| Bypass confusion | Users on wrong subnet get wrong path |
| Conflicts with PCC | Overlapping marks cause unpredictable routing |
| No dynamic adaptation | Static rules don't respond to WAN degradation |

---

## Common Errors

| Error | Fix |
|-------|-----|
| Policy rules below PCC rules | Move policy rules to top of mangle |
| Same connection double-marked | Use `connection-mark=no-mark` on PCC only |
| Policy route without NAT match | Align NAT out-interface with policy WAN |
| Forgetting passthrough=yes | Only first packet gets mark |

---

**Next →** [Recursive Routing](recursive-routing.md)
