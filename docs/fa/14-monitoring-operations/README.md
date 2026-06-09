# فصل ۱۴ — مانیتورینگ و عملیات

> SNMP، Syslog، Netwatch، هشدار ایمیل و داشبوردهای عملیاتی برای Multi-WAN.

---

## ۱۴.۱ معماری مانیتورینگ

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

## ۱۴.۲ معیارهای کلیدی برای مانیتور

| معیار | آستانه | ابزار |
|-------|--------|-------|
| استفاده پهنای باند به‌ازای هر WAN | > 80٪ پایدار | SNMP / آمار Interface |
| دسترسی Gateway | هر DOWN | Netwatch / SNMP |
| وضعیت Route فعال/غیرفعال | هر inactive | Script / SNMP |
| استفاده جدول Connection | > 70٪ max-entries | SNMP |
| استفاده CPU | > 60٪ پایدار | SNMP |
| نسبت توزیع PCC | انحراف > 20٪ | Script |
| نرخ Hit کش DNS | < 50٪ | /ip dns cache print |
| Packet loss به‌ازای هر WAN | > 1٪ | /ping count=100 |

---

## ۱۴.۳ پیکربندی SNMP

```
/snmp
set enabled=yes contact="NOC@company.com" location="HQ-DC1"

/snmp community
add name=monitoring addresses=192.168.1.0/24 security=authorized \
    read-access=yes write-access=no
```

### OIDهای مفید

| OID | توضیح |
|-----|-------|
| `1.3.6.1.2.1.2.2.1.10` | ifInOctets (bytes in) |
| `1.3.6.1.2.1.2.2.1.16` | ifOutOctets (bytes out) |
| `1.3.6.1.2.1.2.2.1.8` | ifOperStatus (up/down) |

---

## ۱۴.۴ Syslog — لاگ از راه دور

```
/system logging action
add name=remote-syslog target=remote remote=192.168.1.200 remote-port=514

/system logging
add topics=route,warning action=remote-syslog
add topics=firewall,warning action=remote-syslog
add topics=critical action=remote-syslog
```

---

## ۱۴.۵ هشدار ایمیل هنگام قطع WAN

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

## ۱۴.۶ اسکریپت تأیید تعادل PCC

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

## ۱۴.۷ پشتیبان خودکار Config

```
/system script
add name=backup-config source={
    /export file=("backup-" . [/system clock get date])
}

/system scheduler
add name=daily-backup interval=1d on-event=backup-config start-time=02:00:00
```

---

## ۱۴.۸ داشبورد ترافیک Interface (CLI)

```
:foreach i in=[/interface find where name~"ether"] do={
    /interface monitor-traffic $i once
}
```

---

## ۱۴.۹ Runbook عملیاتی

| رویداد | اقدام | دستور |
|-------|-------|-------|
| WAN1 قطع | تأیید Failover فعال | `/ip route print detail` |
| PCC نامتعادل | بررسی آمار mangle | `/ip firewall mangle print stats` |
| CPU بالا | بررسی تعداد Connection | `/ip firewall connection print count-only` |
| مشکل NAT | بازرسی Connectionها | `/ip firewall connection print` |
| اینترنت کند | تست Latency به‌ازای هر WAN | `/ping 8.8.8.8 interface=ether1` |
| پس از قطع برق | تأیید فعال بودن تمام Routeها | `/ip route print where inactive` |

---

**فصل بعد ←** [فصل ۱۵: انواع WAN (PPPoE، LTE، DHCP)](../15-wan-types/README.md)
