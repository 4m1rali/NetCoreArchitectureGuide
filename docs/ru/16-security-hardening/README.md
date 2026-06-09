# Глава 16 — Security Hardening

> Advanced firewall, DDoS mitigation и access control для Multi-WAN edge routers.

---

## 16.1 Defense-in-Depth Model

```
Layer 1: Physical — Router in locked rack, console protected
Layer 2: Network — Disable unused services, management VLAN
Layer 3: Firewall — Stateful filter, address lists, RAW chain
Layer 4: Application — VPN for management, no plain Winbox on WAN
Layer 5: Monitoring — Syslog, SNMP traps, connection tracking alerts
```

---

## 16.2 Service Hardening

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

## 16.3 RAW Chain — Pre-Connection Dropping

Drop invalid and known-bad traffic before connection tracking (saves CPU):

```
/ip firewall raw
add chain=prerouting in-interface-list=WAN connection-state=invalid action=drop
add chain=prerouting in-interface-list=WAN src-address-list=blocked-ips action=drop
```

---

## 16.4 Address Lists for Threat Management

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

## 16.5 DDoS Mitigation

```
/ip firewall filter
add chain=input in-interface-list=WAN protocol=tcp connection-limit=30,32 \
    action=drop comment="SYN flood protection"
add chain=forward in-interface-list=WAN protocol=tcp connection-limit=100,32 \
    action=drop comment="Per-source connection limit"
```

### Detect SYN Flood

```
/ip firewall filter
add chain=input protocol=tcp tcp-flags=syn connection-state=new \
    connection-limit=50,32 action=add-src-to-address-list \
    address-list=syn-flooders address-list-timeout=1h

/ip firewall filter
add chain=input src-address-list=syn-flooders action=drop
```

---

## 16.6 Anti-Spoofing

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

## 16.7 Geo-Blocking (Optional)

```
/ip firewall address-list
add list=geo-block address=0.0.0.0/0 comment="placeholder"

# Use external script to populate geo-block list from IP country database
/ip firewall raw
add chain=prerouting in-interface-list=WAN src-address-list=geo-block action=drop
```

---

## 16.8 Management Access via VPN Only

```
/ip firewall filter
add chain=input in-interface-list=WAN protocol=tcp dst-port=22,8291,8728 action=drop \
    comment="Block management on WAN"
add chain=input in-interface=wg-mgmt protocol=tcp dst-port=22,8291 action=accept \
    comment="Management via VPN only"
```

---

## 16.9 Logging and Audit

```
/ip firewall filter
add chain=input in-interface-list=WAN action=log log-prefix="WAN-INPUT-DROP"
add chain=forward in-interface-list=WAN connection-state=new action=log \
    log-prefix="WAN-FWD-NEW"
```

Отключите logging в production после tuning — high volume impacts performance.

---

## 16.10 Security Checklist

| # | Пункт | Статус |
|---|-------|--------|
| 1 | Default admin password changed | ☐ |
| 2 | Unused services disabled | ☐ |
| 3 | Winbox/SSH restricted to LAN/VPN | ☐ |
| 4 | WAN input dropped by default | ☐ |
| 5 | Anti-spoofing rules active | ☐ |
| 6 | Bogon filtering on WAN | ☐ |
| 7 | rp-filter=strict enabled | ☐ |
| 8 | tcp-syncookies enabled | ☐ |
| 9 | SYN flood protection active | ☐ |
| 10 | Config backup encrypted/offsite | ☐ |

---

**Продолжить →** [Глава 25: Security Labs & CVEs](../25-security-labs-cve/README.md)

**Далее →** [Глава 17: Case Studies](../17-case-studies/README.md)
