# فصل ۱۶ — سخت‌سازی امنیتی

> Firewall پیشرفته، کاهش DDoS و کنترل دسترسی برای روترهای لبه Multi-WAN.

---

## ۱۶.۱ مدل Defense-in-Depth

```
Layer 1: Physical — Router in locked rack, console protected
Layer 2: Network — Disable unused services, management VLAN
Layer 3: Firewall — Stateful filter, address lists, RAW chain
Layer 4: Application — VPN for management, no plain Winbox on WAN
Layer 5: Monitoring — Syslog, SNMP traps, connection tracking alerts
```

---

## ۱۶.۲ سخت‌سازی سرویس‌ها

```
/ip service
set telnet disabled=yes
set ftp disabled=yes
set www disabled=yes
set www-ssl disabled=yes
set api disabled=yes
set api-ssl disabled=yes
set winbox address=192.168.1.0/24
set ssh address=192.168.1.0/24

/user group
set read policy=local,telnet,ssh,ftp,reboot,read,test,winbox,password,web,sniff,sensitive,api,romon,dude,tikapp

/user
add name=admin group=full password=strong-password-here
```

---

## ۱۶.۳ زنجیره RAW — Drop قبل از Connection

ترافیک Invalid و شناخته‌شده بد را قبل از Connection Tracking حذف کنید (صرفه‌جویی CPU):

```
/ip firewall raw
add chain=prerouting in-interface-list=WAN connection-state=invalid action=drop
add chain=prerouting in-interface-list=WAN src-address-list=blocked-ips action=drop
```

---

## ۱۶.۴ Address List برای مدیریت تهدید

```
/ip firewall address-list
add list=blocked-ips address=0.0.0.0/8 comment="Bogon"
add list=blocked-ips address=10.0.0.0/8 comment="Bogon"
add list=blocked-ips address=127.0.0.0/8 comment="Bogon"
add list=blocked-ips address=169.254.0.0/16 comment="Bogon"
add list=blocked-ips address=172.16.0.0/12 comment="Bogon"
add list=blocked-ips address=192.168.0.0/16 comment="Bogon"
add list=blocked-ips address=224.0.0.0/4 comment="Multicast"
add list=blocked-ips address=240.0.0.0/4 comment="Reserved"

/ip firewall raw
add chain=prerouting in-interface-list=WAN src-address-list=blocked-ips action=drop
```

---

## ۱۶.۵ کاهش DDoS

```
/ip firewall filter
add chain=input in-interface-list=WAN protocol=tcp connection-limit=30,32 \
    action=drop comment="SYN flood protection"
add chain=forward in-interface-list=WAN protocol=tcp connection-limit=100,32 \
    action=drop comment="Per-source connection limit"
```

### تشخیص SYN Flood

```
/ip firewall filter
add chain=input protocol=tcp tcp-flags=syn connection-state=new \
    connection-limit=50,32 action=add-src-to-address-list \
    address-list=syn-flooders address-list-timeout=1h

/ip firewall filter
add chain=input src-address-list=syn-flooders action=drop
```

---

## ۱۶.۶ Anti-Spoofing

```
/ip firewall filter
add chain=forward in-interface-list=WAN src-address=192.168.0.0/16 action=drop \
    comment="RFC1918 from WAN"
add chain=forward in-interface-list=WAN src-address=10.0.0.0/8 action=drop
add chain=forward in-interface-list=WAN src-address=172.16.0.0/12 action=drop

/ip settings
set rp-filter=strict
set tcp-syncookies=yes
```

---

## ۱۶.۷ Geo-Blocking (اختیاری)

```
/ip firewall address-list
add list=geo-block address=0.0.0.0/0 comment="placeholder"

# Use external script to populate geo-block list from IP country database
/ip firewall raw
add chain=prerouting in-interface-list=WAN src-address-list=geo-block action=drop
```

---

## ۱۶.۸ دسترسی مدیریت فقط از طریق VPN

```
/ip firewall filter
add chain=input in-interface-list=WAN protocol=tcp dst-port=22,8291,8728 action=drop \
    comment="Block management on WAN"
add chain=input in-interface=wg-mgmt protocol=tcp dst-port=22,8291 action=accept \
    comment="Management via VPN only"
```

---

## ۱۶.۹ لاگ و Audit

```
/ip firewall filter
add chain=input in-interface-list=WAN action=log log-prefix="WAN-INPUT-DROP"
add chain=forward in-interface-list=WAN connection-state=new action=log \
    log-prefix="WAN-FWD-NEW"
```

پس از تنظیم، لاگ را در Production غیرفعال کنید — حجم بالا بر کارایی تأثیر می‌گذارد.

---

## ۱۶.۱۰ چک‌لیست امنیتی

| # | مورد | وضعیت |
|---|------|-------|
| 1 | رمز پیش‌فرض admin تغییر کرد | ☐ |
| 2 | سرویس‌های بلااستفاده غیرفعال شد | ☐ |
| 3 | Winbox/SSH به LAN/VPN محدود شد | ☐ |
| 4 | ورودی WAN به‌صورت پیش‌فرض Drop می‌شود | ☐ |
| 5 | قوانین Anti-spoofing فعال است | ☐ |
| 6 | فیلتر Bogon روی WAN | ☐ |
| 7 | rp-filter=strict فعال است | ☐ |
| 8 | tcp-syncookies فعال است | ☐ |
| 9 | محافظت SYN flood فعال است | ☐ |
| 10 | پشتیبان Config رمزگذاری/Offsite | ☐ |

---

**مرتبط ←** [فصل ۲۵: آزمایشگاه امنیت، CVEها و تهدیدهای نادیده‌گرفته‌شده](../25-security-labs-cve/README.md)

**بعدی ←** [فصل ۱۷: مطالعات موردی](../17-case-studies/README.md)
