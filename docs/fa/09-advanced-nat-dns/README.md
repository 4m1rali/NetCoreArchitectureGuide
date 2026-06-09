# فصل ۹ — NAT و DNS پیشرفته

> Hairpin NAT، Load Balancing ورودی، استراتژی‌های DNS و Port Forwarding در مقیاس.

---

## ۹.۱ Hairpin NAT (NAT Loopback)

به کلاینت‌های LAN اجازه می‌دهد سرورهای داخلی را با **IP عمومی WAN** از داخل شبکه در دسترس بگیرند.

### مشکل

```
LAN client → 203.0.113.2 (public IP of web server)
  → Router receives on LAN
  → No dst-nat rule matches (packet from LAN, not WAN)
  → Connection fails
```

### راه‌حل

```
/ip firewall nat
add chain=dstnat in-interface-list=LAN dst-address=203.0.113.2 protocol=tcp \
    dst-port=80,443 action=dst-nat to-addresses=192.168.1.10

add chain=srcnat in-interface-list=LAN dst-address=192.168.1.10 \
    action=masquerade comment="Hairpin NAT"
```

### Hairpin به‌ازای هر WAN

```
add chain=dstnat in-interface-list=LAN dst-address=203.0.113.2 action=dst-nat to-addresses=192.168.1.10
add chain=dstnat in-interface-list=LAN dst-address=198.51.100.2 action=dst-nat to-addresses=192.168.1.10
add chain=srcnat in-interface-list=LAN dst-address=192.168.1.10 action=masquerade
```

---

## ۹.۲ Load Balancing ورودی (DNAT Failover)

ترافیک ورودی را نمی‌توان با PCC متعادل کرد (مسیریابی ISP مسیر Ingress را تعیین می‌کند). برای توزیع ورودی از **DNS round-robin** یا **BGP** استفاده کنید.

### DNS Round-Robin

```
# DNS zone on router or external DNS server
web.example.com  A  203.0.113.2   (ISP-1)
web.example.com  A  198.51.100.2  (ISP-2)
```

### Failover ورودی Active-Passive

WAN اصلی را مانیتور کنید؛ هنگام Failover با اسکریپت TTL DNS را به‌صورت پویا به‌روزرسانی کنید.

```
/tool netwatch
add host=203.0.113.1 interval=10s timeout=3s \
    down-script={
        /ip firewall nat disable [find comment="DNAT ISP-1"]
        /ip firewall nat enable [find comment="DNAT ISP-2 backup"]
    } \
    up-script={
        /ip firewall nat enable [find comment="DNAT ISP-1"]
        /ip firewall nat disable [find comment="DNAT ISP-2 backup"]
    }
```

---

## ۹.۳ استراتژی‌های DNS در Multi-WAN

| استراتژی | پیکربندی | مزایا | معایب |
|----------|----------|-------|-------|
| DNS Cache روتر | `/ip dns set allow-remote-requests=yes` | یک Resolver برای LAN | روتر تبدیل به DNS SPOF می‌شود |
| DNS به‌ازای هر WAN | DHCP client با use-peer-dns روی هر WAN | Resolution بهینه ISP | پاسخ‌های ناهماهنگ |
| DNS عمومی | servers=8.8.8.8,1.1.1.1 | سریع و یکنواخت | بدون بهینه‌سازی CDN ISP |
| Split DNS | Zone داخلی روی روتر، خارجی از WAN | بهترین برای Enterprise | پیچیده |

### DNS پیشنهادی Production

```
/ip dns
set allow-remote-requests=yes cache-size=4096KiB \
    servers=8.8.8.8,1.1.1.1,max-cache-ttl=1w

/ip dhcp-server network
set [find] dns-server=192.168.1.1
```

### مشکل DNS روی PCC

Queryهای DNS Connectionهای UDP کوتاه‌عمر هستند — ممکن است به WANهای مختلف بروند و Endpointهای CDN متفاوت بگیرند.

**راه‌حل:** DNS را فقط از طریق Cache روتر عبور دهید (کلاینت‌ها از 192.168.1.1 استفاده کنند، هرگز مستقیماً DNS خارجی).

```
/ip firewall filter
add chain=forward in-interface-list=LAN protocol=udp dst-port=53 \
    dst-address=!192.168.1.1 action=drop comment="Force DNS via router"
```

---

## ۹.۴ Port Forwarding در مقیاس

### قالب به‌ازای هر سرویس

```
/ip firewall nat
add chain=dstnat in-interface=ether1 protocol=tcp dst-port=443 \
    action=dst-nat to-addresses=192.168.1.10 to-ports=443 comment="HTTPS ISP-1"
add chain=dstnat in-interface=ether2 protocol=tcp dst-port=443 \
    action=dst-nat to-addresses=192.168.1.10 to-ports=443 comment="HTTPS ISP-2"
```

### Port Range Forwarding

```
add chain=dstnat in-interface=ether1 protocol=tcp dst-port=50000-50100 \
    action=dst-nat to-addresses=192.168.1.20 to-ports=50000-50100 comment="RTP ISP-1"
```

### DNAT مبتنی بر Address List

```
/ip firewall address-list
add list=public-services address=192.168.1.10
add list=public-services address=192.168.1.20

/ip firewall nat
add chain=dstnat in-interface-list=WAN protocol=tcp dst-port=80,443 \
    action=dst-nat to-addresses=192.168.1.10
```

---

## ۹.۵ پیشگیری از تداخل NAT

| قانون | پیاده‌سازی |
|-------|------------|
| یک masquerade به‌ازای هر out-interface | قوانین srcnat جداگانه به‌ازای هر WAN |
| هرگز masquerade سراسری | `out-interface-list=WAN` به‌تنهایی کافی نیست — per-interface استفاده کنید |
| هم‌راستایی NAT با Connection Mark | mark → route → interface → زنجیره NAT را تأیید کنید |
| غیرفعال‌سازی قوانین NAT بلااستفاده | قوانین غیرفعال هنوز Audit را گیج می‌کنند |
| لاگ NAT miss | `action=log` روی قانون catch-all srcnat (فقط Debug) |

---

## ۹.۶ Src-NAT با IP عمومی ثابت

وقتی ISP بلوک IP عمومی ثابت ارائه می‌دهد:

```
/ip firewall nat
add chain=srcnat out-interface=ether1 action=src-nat \
    to-addresses=203.0.113.2 comment="Fixed NAT ISP-1"
add chain=srcnat out-interface=ether2 action=src-nat \
    to-addresses=198.51.100.2 comment="Fixed NAT ISP-2"
```

Src-NAT ثابت سرویس‌های ورودی را بدون تداخل نگاشت پورت پویا ممکن می‌سازد.

---

**فصل بعد ←** [فصل ۱۰: QoS و اولویت‌بندی ترافیک](../10-qos-traffic/README.md)
