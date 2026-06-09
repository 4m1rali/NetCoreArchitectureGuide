# Recursive Routing

> Gateway reachability resolution — foundation of reliable Multi-WAN.

---

## Engineering Definition

**Recursive Routing** is the process where a router resolves the reachability of a gateway address through another route, rather than assuming the gateway is directly connected. In MikroTik, this uses **scope** and **target-scope** attributes on routes.

Without recursive routing, `check-gateway` cannot verify gateway health on non-directly-connected WAN links (PPPoE, metro Ethernet, VLAN handoff).

---

## Scope and Target-Scope Explained

| Attribute | Value | Meaning |
|-----------|-------|---------|
| `scope` | 10 | Route used to resolve gateway reachability |
| `target-scope` | default (30) | Route used for actual packet forwarding |
| `scope` | 30 | Directly connected — used for forwarding |

### Resolution Flow

```
Default route: 0.0.0.0/0 via 203.0.113.1 (target-scope=30)
  → Router must find how to reach 203.0.113.1
  → Host route: 203.0.113.1/32 via ether1 (scope=10)
  → 203.0.113.1 reachable via ether1
  → Default route: ACTIVE
  → check-gateway pings 203.0.113.1 via ether1
```

---

## Configuration Patterns

### Static IP WAN (Directly Connected)

```
/ip route
add dst-address=203.0.113.1/32 gateway=ether1 scope=10
add dst-address=0.0.0.0/0 gateway=203.0.113.1 distance=1 check-gateway=ping
```

### PPPoE WAN (Interface as Gateway)

```
/interface pppoe-client
add name=pppoe-isp1 interface=ether1 user=isp1@provider password=secret

/ip route
add dst-address=0.0.0.0/0 gateway=pppoe-isp1 distance=1 check-gateway=ping
```

### Multi-Hop Metro Ethernet

```
/ip route
add dst-address=198.51.100.1/32 gateway=10.0.0.1 scope=10
add dst-address=10.0.0.0/24 gateway=ether2
add dst-address=0.0.0.0/0 gateway=198.51.100.1 distance=1 check-gateway=ping
```

### Recursive with Multiple Monitored Targets

```
/ip route
add dst-address=203.0.113.1/32 gateway=ether1 scope=10
add dst-address=8.8.8.8/32 gateway=203.0.113.1 scope=10 target-scope=10
add dst-address=0.0.0.0/0 gateway=8.8.8.8 distance=1 check-gateway=ping
```

This monitors internet reachability, not just gateway L2 reachability.

---

## Use Cases

| Scenario | Why Recursive |
|----------|--------------|
| PPPoE WAN | Gateway is virtual interface, not IP |
| Metro Ethernet | Gateway behind provider subnet |
| VLAN WAN handoff | Gateway on different VLAN subnet |
| GRE tunnel WAN | Gateway inside tunnel |
| ISP with /30 subnet | Gateway not in connected route |

---

## Pros

| Advantage | Detail |
|-----------|--------|
| Reliable check-gateway | Works on any WAN handoff type |
| Internet-level monitoring | Can monitor 8.8.8.8 instead of just gateway |
| Flexible topology | Supports complex ISP handoff designs |
| Failover accuracy | Dead routes removed when gateway truly unreachable |

---

## Cons and Risks

| Risk | Detail |
|------|--------|
| Misconfigured scope | Route never becomes active |
| Circular resolution | Gateway route points to itself |
| Extra routes needed | More config than directly-connected |
| Debugging complexity | Route print detail required to diagnose |

---

## Diagnostic Commands

```
/ip route print detail where dst-address=0.0.0.0/0
/ip route print detail where dst-address~"203.0.113"
/ip route check 203.0.113.1
```

---

**Next →** [Connection Tracking Deep Dive](connection-tracking-deep.md)
