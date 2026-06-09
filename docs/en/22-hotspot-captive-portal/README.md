# Chapter 22 — Hotspot & Captive Portal on Multi-WAN

> Guest WiFi authentication with PCC load balancing.

---

## 22.1 Hotspot + Multi-WAN Architecture

```
Guest WiFi (VLAN 10) → Hotspot → PCC → WAN1/WAN2/WAN3
Corporate LAN (VLAN 1) → Policy Routing → WAN1 (primary)
```

Isolate guest traffic in a separate VLAN with its own PCC path.

---

## 22.2 Hotspot Configuration

```
/interface bridge
add name=bridge-hotspot

/interface vlan
add name=vlan-guest vlan-id=10 interface=bridge-lan

/interface bridge port
add bridge=bridge-hotspot interface=vlan-guest

/ip pool
add name=hotspot-pool ranges=10.10.10.2-10.10.10.254

/ip hotspot profile
add name=hsprof hotspot-address=10.10.10.1 dns-name=login.wifi.local

/ip hotspot
add name=hotspot1 interface=bridge-hotspot address-pool=hotspot-pool profile=hsprof

/ip hotspot user
add name=guest password=guest1 profile=default
```

---

## 22.3 Hotspot PCC Integration

```
/ip firewall mangle
add chain=prerouting in-interface=bridge-hotspot connection-mark=no-mark \
    dst-address-type=!local per-connection-classifier=both-addresses-and-ports:2/0 \
    action=mark-connection new-connection-mark=WAN2-conn passthrough=yes
add chain=prerouting in-interface=bridge-hotspot connection-mark=no-mark \
    dst-address-type=!local per-connection-classifier=both-addresses-and-ports:2/1 \
    action=mark-connection new-connection-mark=WAN3-conn passthrough=yes
```

Guest traffic balanced across WAN2 + WAN3; corporate stays on WAN1.

---

## 22.4 Hotspot Rate Limiting

```
/ip hotspot user profile
set default rate-limit=5M/20M shared-users=3
add name=premium rate-limit=20M/50M
```

---

## 22.5 Walled Garden (Allow Without Auth)

```
/ip hotspot walled-garden
add dst-host=*.google.com
add dst-host=*.apple.com
add dst-host=captive.apple.com
```

---

**Next Chapter →** [Chapter 23: Automation & Scripting](../23-automation-scripting/README.md)
