# راهنمای معماری Enterprise چند-WAN میکروتیک

> **معمار ارشد شبکه ISP · متخصص MikroTik · طراح شبکه Enterprise**
>
> مستندات آماده Production برای استقرار Multi-WAN در ISP / Enterprise / Datacenter روی MikroTik RouterOS.

---

## انتخاب زبان / Language Selection / Выбор языка

| زبان | کد | مسیر مستندات |
|------|-----|--------------|
| **English** | `EN` | [docs/en/](docs/en/) |
| **فارسی (Persian)** | `FA` | [docs/fa/](docs/fa/) |
| **Русский (Russian)** | `RU` | [docs/ru/](docs/ru/) |

---

## خلاصه اجرایی

این راهنما یک کتاب مهندسی کامل و فصل‌به‌فصل برای طراحی، استقرار و بهره‌برداری از شبکه‌های **Multi-WAN روی MikroTik RouterOS** در مقیاس Production است.

پس از مطالعه این مستندات، قادر خواهید بود:

- یک توپولوژی **Multi-WAN** واقعی با ۲ تا ۳ ISP طراحی و استقرار دهید
- **Load Balancing**، **Failover**، **ECMP** و **PCC** را در سطح Production پیاده‌سازی کنید
- **NAT**، **Routing** و **Firewall** را به‌صورت یک سیستم یکپارچه مدیریت کنید
- ناپایداری، خطاهای مسیریابی و تداخل NAT را در شبکه‌های زنده عیب‌یابی کنید
- ترکیب صحیح روش‌ها را برای سناریوهای ISP، Enterprise یا Datacenter انتخاب کنید

---

## ساختار کتاب (فصل‌به‌فصل)

```
NetCoreArchitectureGuide/
├── README.md                          ← فهرست انگلیسی + انتخاب زبان
└── docs/
    ├── en/                            ← English (full content)
    ├── fa/                            ← فارسی (شما اینجا هستید)
    └── ru/                            ← Русский
        ├── 01-network-architecture/
        ├── 02-core-concepts/
        │   ├── ecmp.md
        │   ├── pcc.md
        │   ├── failover.md
        │   ├── load-balancing.md
        │   ├── policy-routing.md
        │   ├── recursive-routing.md
        │   └── connection-tracking-deep.md
        ├── 03-comparison-table/
        ├── 04-production-design/
        ├── 05-mikrotik-configuration/
        ├── 06-debugging-troubleshooting/
        ├── 07-engineering-analysis/
        ├── 08-final-summary/
        ├── 09-advanced-nat-dns/
        ├── 10-qos-traffic/
        ├── 11-vpn-multiwan/
        ├── 12-ipv6-multiwan/
        ├── 13-bgp-multihoming/
        ├── 14-monitoring-operations/
        ├── 15-wan-types/
        ├── 16-security-hardening/
        ├── 17-case-studies/
        ├── 18-routeros-licensing/
        ├── 19-hardware-selection/
        ├── 20-migration-upgrade/
        ├── 21-high-availability/
        ├── 22-hotspot-captive-portal/
        ├── 23-automation-scripting/
        ├── 24-disaster-recovery/
        ├── 25-security-labs-cve/
        └── appendix/
```

---

## فهرست فصل‌ها (فارسی)

### بخش اول — مبانی

| # | فصل | توضیح | لینک |
|---|-----|-------|------|
| 1 | **معماری شبکه** | طراحی Multi-WAN، نقش NAT، جداول مسیریابی، Connection Tracking، L2/L3/L4، جریان بسته | [مطالعه ←](01-network-architecture/README.md) |
| 2 | **مفاهیم پایه** | مهندسی عمیق: ECMP، PCC، Failover، Load Balancing، Policy Routing | [مطالعه ←](02-core-concepts/README.md) |
| 3 | **جدول مقایسه** | ماتریس مقایسه روش‌ها در سطح Enterprise | [مطالعه ←](03-comparison-table/README.md) |

### بخش دوم — طراحی و پیکربندی

| # | فصل | توضیح | لینک |
|---|-----|-------|------|
| 4 | **طراحی Production** | سناریوی واقعی ۳ ISP، توپولوژی، جریان ترافیک، مسیر NAT | [مطالعه ←](04-production-design/README.md) |
| 5 | **پیکربندی MikroTik** | اسکریپت‌های آماده RouterOS: WAN، PCC، Failover، ECMP، NAT، Firewall | [مطالعه ←](05-mikrotik-configuration/README.md) |

### بخش سوم — عملیات

| # | فصل | توضیح | لینک |
|---|-----|-------|------|
| 6 | **عیب‌یابی و Troubleshooting** | تحلیل خرابی‌های واقعی و ابزارهای Debug میکروتیک | [مطالعه ←](06-debugging-troubleshooting/README.md) |
| 7 | **تحلیل مهندسی** | دیدگاه متخصص: چه زمانی ECMP، PCC، Failover و استراتژی‌های ترکیبی | [مطالعه ←](07-engineering-analysis/README.md) |
| 8 | **خلاصه نهایی معماری** | توصیه‌های Best Practice برای طراحی Multi-WAN | [مطالعه ←](08-final-summary/README.md) |

### بخش چهارم — موضوعات پیشرفته

| # | فصل | توضیح | لینک |
|---|-----|-------|------|
| 9 | **NAT و DNS پیشرفته** | Hairpin NAT، Load Balancing ورودی، استراتژی DNS | [مطالعه ←](09-advanced-nat-dns/README.md) |
| 10 | **QoS و ترافیک** | مدیریت پهنای باند و اولویت‌بندی | [مطالعه ←](10-qos-traffic/README.md) |
| 11 | **VPN روی Multi-WAN** | IPsec، WireGuard، L2TP | [مطالعه ←](11-vpn-multiwan/README.md) |
| 12 | **IPv6 Multi-WAN** | Dual-stack و IPv6-only | [مطالعه ←](12-ipv6-multiwan/README.md) |
| 13 | **BGP Multi-Homing** | اتصال چند-خانه ISP-grade | [مطالعه ←](13-bgp-multihoming/README.md) |
| 14 | **مانیتورینگ و عملیات** | SNMP، Syslog، Netwatch، NOC | [مطالعه ←](14-monitoring-operations/README.md) |
| 15 | **انواع WAN** | PPPoE، LTE، DHCP | [مطالعه ←](15-wan-types/README.md) |
| 16 | **سخت‌سازی امنیتی** | Firewall پیشرفته، DDoS | [مطالعه ←](16-security-hardening/README.md) |
| 17 | **مطالعات موردی** | استقرارهای واقعی Production | [مطالعه ←](17-case-studies/README.md) |

### بخش پنجم — مرجع و عملیات

| # | فصل | توضیح | لینک |
|---|-----|-------|------|
| 18 | **لایسنس RouterOS** | سطوح لایسنس، Featureها، سازگاری Multi-WAN | [مطالعه ←](18-routeros-licensing/README.md) |
| 19 | **انتخاب سخت‌افزار** | ماتریس سخت‌افزار، Sizing، RAM و CPU | [مطالعه ←](19-hardware-selection/README.md) |
| 20 | **Migration و Upgrade** | RouterOS 6 → 7، Rollback، Zero-Downtime | [مطالعه ←](20-migration-upgrade/README.md) |
| 21 | **High Availability** | VRRP، همگام‌سازی Config، BGP HA | [مطالعه ←](21-high-availability/README.md) |
| 22 | **Hotspot و Captive Portal** | WiFi مهمان + PCC، Rate Limit | [مطالعه ←](22-hotspot-captive-portal/README.md) |
| 23 | **Automation و Scripting** | Script، Scheduler، REST API | [مطالعه ←](23-automation-scripting/README.md) |
| 24 | **Disaster Recovery** | Backup، RTO/RPO، Spare Hardware | [مطالعه ←](24-disaster-recovery/README.md) |
| 25 | **آزمایشگاه امنیت و CVE** | تمرین‌های عملی، مرجع آسیب‌پذیری، تهدیدهای نادیده‌گرفته‌شده | [مطالعه ←](25-security-labs-cve/README.md) |

### پیوست

| سند | لینک |
|-----|------|
| واژه‌نامه | [appendix/glossary.md](appendix/glossary.md) |
| برگه مرجع سریع | [appendix/cheat-sheet.md](appendix/cheat-sheet.md) |
| سؤالات متداول (FAQ) | [appendix/faq.md](appendix/faq.md) |
| اشتباهات رایج | [appendix/common-mistakes.md](appendix/common-mistakes.md) |
| راهنمای Wireless و LTE | [appendix/wireless-lte-guide.md](appendix/wireless-lte-guide.md) |

---

## مخاطبان هدف

| نقش | کاربرد |
|-----|--------|
| مهندس شبکه ISP | Multi-homed upstream، توزیع ترافیک، Failover با SLA |
| مدیر شبکه Enterprise | WAN دوگانه/سه‌گانه برای شعب و دفتر مرکزی |
| اپراتور Datacenter | تنوع مسیر خروجی و افزونگی Gateway |
| مشاور MikroTik | قالب‌های آماده Production برای مشتری |

---

## نمای کلی سناریو

```
                    ┌─────────────┐
                    │   ISP-1     │
                    └──────┬──────┘
                           │ WAN1 (ether1)
┌──────────┐         ┌─────┴──────┐         ┌─────────────┐
│  LAN     │─────────│  MikroTik  │─────────│   ISP-2     │
│ Clients  │  ether5 │   CCR/RB   │ WAN2    └─────────────┘
│ Servers  │         │   Router   │ ether2
└──────────┘         │            │─────────┌─────────────┐
                     │            │ WAN3    │   ISP-3     │
                     └────────────┘ ether3  └─────────────┘
```

**روش‌های پوشش‌داده‌شده:**

| روش | لایه | هدف |
|-----|------|-----|
| PCC | L3/L4 (Mangle + Routing) | توزیع بار به‌ازای هر Connection |
| ECMP | L3 (Routing) | مسیریابی Equal-Cost Multipath |
| Failover | L3 (Gateway monitoring) | Fallback خودکار WAN |
| NAT | L3/L4 | ترجمه آدرس برای هر مسیر WAN |

---

## مسیر شروع سریع

1. [فصل ۱ — معماری شبکه](01-network-architecture/README.md) را بخوانید تا جریان بسته و Connection Tracking را درک کنید.
2. [فصل ۲ — مفاهیم پایه](02-core-concepts/README.md) را برای جزئیات داخلی ECMP، PCC، Failover و Load Balancing مطالعه کنید.
3. [فصل ۳ — جدول مقایسه](03-comparison-table/README.md) را مرور کنید تا روش مناسب را انتخاب کنید.
4. [فصل ۴ — طراحی Production](04-production-design/README.md) را برای توپولوژی مرجع دنبال کنید.
5. [فصل ۵ — پیکربندی MikroTik](05-mikrotik-configuration/README.md) را مستقیماً روی روتر اعمال کنید.
6. [فصل ۶ — عیب‌یابی](06-debugging-troubleshooting/README.md) را در زمان Go-Live در دسترس داشته باشید.
7. تصمیمات را با [فصل ۷ — تحلیل مهندسی](07-engineering-analysis/README.md) اعتبارسنجی کنید.
8. با [فصل ۸ — خلاصه](08-final-summary/README.md) نهایی‌سازی کنید.
9. برای موضوعات پیشرفته فصل‌های [۹ تا ۱۷](09-advanced-nat-dns/README.md) را مطالعه کنید.
10. برای مرجع عملیاتی فصل‌های [۱۸ تا ۲۴](18-routeros-licensing/README.md) و [پیوست](appendix/faq.md) را مطالعه کنید.

---

## توصیه Production (خلاصه)

| جزء | روش پیشنهادی |
|-----|--------------|
| توزیع بار | **PCC** (Per Connection Classifier) |
| افزونگی Gateway | **Recursive Routing + check-gateway** |
| مسیر پشتیبان | **Failover** با Gatewayهای مانیتورشده |
| LAN با Throughput بالا | **ECMP** فقط وقتی Session Stickiness لازم نیست |
| NAT | **Masquerade به‌ازای هر WAN** با Connection Mark |
| امنیت | **Stateful Firewall** + قوانین Anti-loop |

> **بهترین ترکیب Production:** PCC (اصلی) + Failover (مانیتورینگ Gateway) + NAT به‌ازای هر WAN + Stateful Firewall

---

## سازگاری RouterOS

| مورد | الزام |
|------|-------|
| نسخه RouterOS | ۷.x توصیه می‌شود (۶.x با یادداشت‌های سازگاری) |
| سخت‌افزار | RB4011، سری CCR یا معادل برای Throughput بالا |
| لایسنس | Level 4+ برای قابلیت‌های پیشرفته Routing |
| اینترفیس‌ها | حداقل ۳ WAN + ۱ LAN |

---

## قراردادهای مستند

- بلوک‌های پیکربندی **آماده Copy-Paste** هستند — بدون توضیح درون اسکریپت
- آدرس‌های IP از محدوده مستندات RFC 5737 استفاده می‌کنند (`203.0.113.0/24`، `198.51.100.0/24`)
- تمام دیاگرام‌ها ASCII هستند برای رندر یکسان در Notion، GitHub و Markdown ساده
- نسخه‌های فارسی و روسی ساختار انگلیسی را دقیقاً منعکس می‌کنند

---

## لایسنس و استفاده

مستندات: **[MIT License](../../LICENSE)** | RouterOS: لایسنس جداگانه دستگاه MikroTik ([فصل ۱۸](18-routeros-licensing/README.md))

توضیح فارسی لایسنس: [LICENSE.fa.md](../../LICENSE.fa.md)

این مستندات برای استفاده حرفه‌ای در مهندسی شبکه تهیه شده است. قبل از استقرار Production، آدرس‌های IP، نام اینترفیس‌ها و پارامترهای ISP را با محیط خود تطبیق دهید.

---

**[شروع مطالعه — فصل ۱: معماری شبکه ←](01-network-architecture/README.md)** | **[FAQ ←](appendix/faq.md)**
