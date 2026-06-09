# Chapter 21 — High Availability (Router Redundancy)

> Eliminating the router as a single point of failure in Multi-WAN designs.

---

## 21.1 HA Architecture Options

| Method | Failover Time | Complexity | Cost |
|--------|--------------|------------|------|
| VRRP (Virtual Router Redundancy) | 1–3 seconds | Medium | 2 routers |
| Dual router manual failover | Minutes | Low | 2 routers |
| BGP with two edge routers | Seconds | High | 2 routers + BGP |
| Cloud CHR failover | Seconds | Medium | 2 VMs |

---

## 21.2 VRRP on MikroTik

```
/interface vrrp
add interface=bridge-lan vrid=1 priority=150 preempt=yes \
    authentication=ah2 password=vrrp-secret

/ip address
add address=192.168.1.1/24 interface=bridge-lan comment="LAN gateway (VRRP virtual)"
```

### Dual Router VRRP Design

```
Router-A (Master)                    Router-B (Backup)
├── Priority: 150                    ├── Priority: 100
├── All WAN links active             ├── All WAN links active
├── PCC + Failover configured        ├── Identical PCC + Failover config
├── VRRP Master                      ├── VRRP Backup
└── Handles traffic                  └── Takes over on Master failure
```

Both routers run identical PCC configuration. VRRP only protects the **LAN gateway IP**.

---

## 21.3 Config Synchronization

Use **configuration export/import** or **The Dude** to keep configs synchronized:

```
# On master — scheduled export to shared storage
/system scheduler
add name=config-sync interval=1h on-event={
    /export file=master-config
    /tool fetch address=192.168.1.200 src-path=master-config.rsc \
        dst-path=backup-config.rsc mode=ftp upload=yes
}
```

---

## 21.4 WAN HA Considerations

| Aspect | Detail |
|--------|--------|
| Both routers need all WAN links | Physical cable to each router or switch between ISP and routers |
| NAT state not synchronized | Active connections drop on VRRP failover (~1–3s) |
| PCC state not synchronized | New connections redistribute after failover |
| BGP | Both routers can peer — use BGP for WAN HA instead of VRRP |
| Connection tracking | Not shared between routers — expected session loss |

---

## 21.5 ISP-Grade HA with BGP

```
Router-A (AS 65050)          Router-B (AS 65050)
├── BGP to ISP-1             ├── BGP to ISP-1
├── BGP to ISP-2             ├── BGP to ISP-2
├── Advertise PI space       ├── Advertise PI space
└── Active                   └── Standby (higher prepend)

ISP sees two paths to your prefix — automatic failover without VRRP.
```

---

**Next Chapter →** [Chapter 22: Hotspot & Captive Portal](../22-hotspot-captive-portal/README.md)
