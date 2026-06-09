# Глава 14 — Monitoring & Operations

> SNMP, Syslog, Netwatch, email alerts и operational dashboards для Multi-WAN.

---

## 14.1 Monitoring Architecture

```
┌─────────────┐     SNMP/SSH     ┌──────────────┐
│  MikroTik   │─────────────────→│  LibreNMS /  │
│  Multi-WAN  │     Syslog       │  Zabbix /    │
│  Router     │─────────────────→│  Grafana     │
└─────────────┘     Netwatch     └──────────────┘
       │                                │
       │  Email/Telegram                ▼
       └──────────────────────→  NOC Dashboard
```

---

## 14.2 Key Metrics to Monitor

| Метрика | Порог | Инструмент |
|---------|-------|------------|
| Per-WAN bandwidth utilization | > 80% sustained | SNMP / interface stats |
| Gateway reachability | Any DOWN | Netwatch / SNMP |
| Route active/inactive state | Any inactive | Script / SNMP |
| Connection table usage | > 70% max-entries | SNMP |
| CPU usage | > 60% sustained | SNMP |
| PCC distribution ratio | Deviation > 20% | Script |
| DNS cache hit rate | < 50% | /ip dns cache print |
| Packet loss per WAN | > 1% | /ping count=100 |

---

## 14.3 SNMP Configuration

```
/snmp
set enabled=yes contact="NOC@company.com" location="HQ-DC1"

/snmp community
add name=monitoring addresses=192.168.1.0/24 security=authorized \
    read-access=yes write-access=no
```

### Useful OIDs

| OID | Описание |
|-----|----------|
| `1.3.6.1.2.1.2.2.1.10` | ifInOctets (bytes in) |
| `1.3.6.1.2.1.2.2.1.16` | ifOutOctets (bytes out) |
| `1.3.6.1.2.1.2.2.1.8` | ifOperStatus (up/down) |

---

## 14.4 Syslog Remote Logging

```
/system logging action
add name=remote-syslog target=remote remote=192.168.1.200 remote-port=514

/system logging
add topics=route,warning action=remote-syslog
add topics=firewall,warning action=remote-syslog
add topics=critical action=remote-syslog
```

---

## 14.5 Email Alerts on WAN Failure

```
/tool e-mail
set server=smtp.company.com port=587 user=alerts@company.com password=secret

/system script
add name=wan-down-alert source={
    /tool e-mail send to="noc@company.com" subject="WAN FAILURE" \
        body="Gateway unreachable - check router immediately"
}

/tool netwatch
add host=203.0.113.1 interval=10s timeout=3s \
    down-script="/system script run wan-down-alert"
```

---

## 14.6 PCC Balance Verification Script

```
/system script
add name=check-pcc-balance source={
    :local w1 [/ip firewall connection print count-only where connection-mark="WAN1-conn"]
    :local w2 [/ip firewall connection print count-only where connection-mark="WAN2-conn"]
    :local w3 [/ip firewall connection print count-only where connection-mark="WAN3-conn"]
    :local total ($w1 + $w2 + $3)
    :log info ("PCC Balance: WAN1=$w1 WAN2=$w2 WAN3=$w3 Total=$total")
}

/system scheduler
add name=pcc-check interval=5m on-event=check-pcc-balance
```

---

## 14.7 Automated Config Backup

```
/system script
add name=backup-config source={
    /export file=("backup-" . [/system clock get date])
}

/system scheduler
add name=daily-backup interval=1d on-event=backup-config start-time=02:00:00
```

---

## 14.8 Interface Traffic Dashboard (CLI)

```
:foreach i in=[/interface find where name~"ether"] do={
    /interface monitor-traffic $i once
}
```

---

## 14.9 Operational Runbook

| Событие | Действие | Команда |
|---------|----------|---------|
| WAN1 down | Verify failover active | `/ip route print detail` |
| Imbalanced PCC | Check mangle stats | `/ip firewall mangle print stats` |
| High CPU | Check connection count | `/ip firewall connection print count-only` |
| NAT issues | Inspect connections | `/ip firewall connection print` |
| Slow internet | Test per-WAN latency | `/ping 8.8.8.8 interface=ether1` |
| After power loss | Verify all routes active | `/ip route print where inactive` |

---

**Следующая глава →** [Глава 15: WAN Types (PPPoE, LTE, DHCP)](../15-wan-types/README.md)
