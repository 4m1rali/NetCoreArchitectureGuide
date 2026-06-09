# Chapter 20 — Migration & Upgrade (RouterOS 6 → 7)

> Safe migration path for existing Multi-WAN deployments upgrading to RouterOS 7.

---

## 20.1 Why Migrate to RouterOS 7

| Feature | ROS 6 | ROS 7 |
|---------|-------|-------|
| Routing tables | `/routing mark` | `/routing table` (cleaner) |
| BGP | Legacy instance model | Template + connection model |
| ECMP per-connection | Not available | `ecmp-per-connection=yes` |
| WireGuard | Limited | Native, full support |
| VRF | Basic | Full VRF support |
| IPv6 firewall | Shared with IPv4 | Separate `/ipv6 firewall` |
| Container | No | Docker-like containers |
| Performance | Good | 20–40% better on same hardware |
| Long-term support | Maintenance mode | Active development |

---

## 20.2 Pre-Migration Checklist

| # | Task | Command |
|---|------|---------|
| 1 | Full config export | `/export file=pre-migration-backup` |
| 2 | Binary backup | `/system backup save name=pre-migration` |
| 3 | Document current routes | `/ip route print detail` |
| 4 | Document mangle rules | `/ip firewall mangle print` |
| 5 | Document NAT rules | `/ip firewall nat print` |
| 6 | Record license level | `/system license print` |
| 7 | Test in lab first | Never upgrade production directly |
| 8 | Schedule maintenance window | 30–60 minutes |

---

## 20.3 Key Configuration Changes

### Routing Tables (Biggest Change)

```
# RouterOS 6
/ip route rule
add src-address=192.168.1.0/24 action=lookup routing-mark=to-WAN1

/routing mark
add name=to-WAN1

/ip route
add dst-address=0.0.0.0/0 gateway=GW1 routing-mark=to-WAN1

# RouterOS 7
/routing table
add name=to-WAN1 fib

/ip route
add dst-address=0.0.0.0/0 gateway=GW1 routing-table=to-WAN1

# Mangle stays the same
add chain=prerouting connection-mark=WAN1-conn action=mark-routing \
    new-routing-mark=to-WAN1 passthrough=yes
```

### BGP (Complete Redesign)

```
# RouterOS 6
/routing bgp instance
/routing bgp peer
/routing bgp network

# RouterOS 7
/routing bgp template
/routing bgp connection
/routing bgp network
```

### Firewall

```
# RouterOS 7 — IPv6 firewall is separate
/ipv6 firewall filter
/ipv6 firewall nat
/ipv6 firewall mangle
```

---

## 20.4 Migration Procedure

```
STEP 1: Lab test with exported config
STEP 2: Upgrade RouterOS (not RouterBOARD firmware yet)
        /system package update check-for-updates
        /system package update download
        /system reboot
STEP 3: Verify auto-migration of config
        /ip route print detail
        /routing table print
STEP 4: Fix any broken rules manually
STEP 5: Test PCC distribution
        /ip firewall mangle print stats
STEP 6: Test failover (disconnect one WAN)
STEP 7: Test NAT
        /ip firewall connection print
STEP 8: Upgrade RouterBOARD firmware if stable
        /system routerboard upgrade
        /system reboot
STEP 9: Monitor for 24 hours before closing maintenance
```

---

## 20.5 Rollback Plan

```
# If migration fails:
1. Netinstall with RouterOS 6.x
2. Import pre-migration backup:
   /import file=pre-migration-backup.rsc
3. Or restore binary backup:
   /system backup load name=pre-migration
```

Keep pre-migration backup on your laptop and a USB drive.

---

## 20.6 Common Migration Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| PCC stops working | routing-mark vs routing-table | Create `/routing table` entries |
| BGP sessions down | New BGP config format | Reconfigure with template model |
| IPv6 firewall missing | Split from IPv4 | Recreate in `/ipv6 firewall` |
| FastTrack breaks PCC | Behavior change | Disable FastTrack |
| Wireless config lost | Wireless package separate | Install wireless package in ROS 7 |
| Scripts fail | Syntax changes | Review and update scripts |

---

## 20.7 Zero-Downtime Migration (Dual Router)

For critical production networks:

```
1. Deploy second router with RouterOS 7 + new config
2. Test thoroughly on separate ports
3. Migrate LAN gateway IP to new router
4. Old router becomes hot standby
5. Keep old router for 30 days as rollback
```

---

**Next Chapter →** [Chapter 21: High Availability](../21-high-availability/README.md)
