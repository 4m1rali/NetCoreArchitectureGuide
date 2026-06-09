# فصل ۲۵ — آزمایشگاه امنیت، CVEها و تهدیدهای نادیده‌گرفته‌شده

> محیط‌های تمرین عملی، مرجع آسیب‌پذیری‌های MikroTik و شکاف‌های امنیتی که در استقرارهای Multi-WAN در Production به‌ندرت پوشش داده می‌شوند.

---

## هدف این فصل

| بخش | مخاطب | هدف |
|-----|-------|-----|
| [تمرین‌های آزمایشگاهی](lab-exercises.md) | مهندسان در حال یادگیری Multi-WAN | تمرین ایمن در آزمایشگاه ایزوله |
| [مرجع CVE و Exploit](cve-exploit-reference.md) | تیم‌های امنیت، NOC | شناخت آسیب‌پذیری‌ها، Patch، تشخیص |
| [امنیت نادیده‌گرفته‌شده](overlooked-security.md) | معماران، ممیزان | بستن شکاف‌هایی که دیگران از قلم می‌اندازند |

> **تذکر اخلاقی:** تمرین‌های آزمایشگاهی و اطلاعات CVE در این فصل **فقط برای سخت‌سازی دفاعی و آموزش** است. هرگز Exploitها را روی شبکه‌های Production یا سیستم‌هایی که مالک آن‌ها نیستید آزمایش نکنید. همیشه در محیط‌های آزمایشگاهی ایزوله کار کنید.

---

## وضعیت امنیتی در بستر Multi-WAN

روترهای Multi-WAN **اهداف باارزش** هستند:

- در لبه اینترنت با چند IP عمومی قرار دارند
- NAT کل سازمان را حمل می‌کنند
- اغلب نسخه‌های قدیمی RouterOS دارند
- سرویس‌های مدیریت (Winbox، API، SSH) به‌طور مکرر در معرض دید قرار دارند
- Scriptها و Netwatch سطح حمله Automation ایجاد می‌کنند

```
                    ATTACK SURFACE MAP
    ┌─────────────────────────────────────────────┐
    │              MikroTik Edge Router              │
    │                                              │
    │  [Winbox:8291]  [SSH:22]  [API:8728]        │ ← Management
    │  [SNMP:161]     [WWW]     [FTP]              │ ← Services
    │  [BGP:179]      [IPsec]   [WireGuard]        │ ← Protocols
    │  [DNS:53]       [NTP]     [Bandwidth-test]   │ ← Auxiliary
    │  [Hotspot]      [Userman] [Container]        │ ← Applications
    │  [Scripts]      [Netwatch] [Scheduler]       │ ← Automation
    │                                              │
    │  WAN1 ──── WAN2 ──── WAN3                   │ ← Multi-WAN
    └─────────────────────────────────────────────┘
```

---

## محتوای فصل

| سند | توضیح |
|-----|-------|
| [تمرین‌های آزمایشگاهی](lab-exercises.md) | ۱۲ آزمایشگاه عملی: PCC، Failover، عیب‌یابی NAT، سخت‌سازی امنیتی |
| [مرجع CVE و Exploit](cve-exploit-reference.md) | CVEهای تاریخی MikroTik، تأثیر، نسخه‌های آسیب‌پذیر، Mitigation |
| [امنیت نادیده‌گرفته‌شده](overlooked-security.md) | بیش از ۲۵ شکاف امنیتی که به‌ندرت در Multi-WAN پوشش داده می‌شوند |

---

## حداقل Baseline امنیتی (قبل از هر آزمایشگاه یا Production)

```
/system package update check-for-updates
/system package update download
/system reboot

/ip service
set telnet disabled=yes
set ftp disabled=yes
set www disabled=yes
set api disabled=yes
set winbox address=192.168.1.0/24
set ssh address=192.168.1.0/24

/user
set admin password="strong-unique-password-min-16-chars"

/ip firewall filter
add chain=input in-interface-list=WAN action=drop place-before=0
```

---

## محیط آزمایشگاهی پیشنهادی

| جزء | مشخصات |
|-----|--------|
| Hypervisor | VMware، Hyper-V یا Proxmox |
| Image روتر | CHR (Cloud Hosted Router) — لایسنس رایگان 1Mbps کافی است |
| نسخه RouterOS | آخرین Stable 7.x + یک نسخه قدیمی 6.x برای مقایسه CVE |
| WANهای مجازی | ۳ شبکه مجازی شبیه‌سازی ISP |
| LAN مجازی | ۱ شبکه با کلاینت‌های تست |
| ایزولاسیون | **بدون اتصال به اینترنت Production** — از DNS/NTP داخلی استفاده کنید |
| Snapshot | قبل از هر آزمایشگاه Snapshot بگیرید برای Rollback فوری |

---

## گردش‌کار ارزیابی امنیتی

```
1. فهرست‌برداری نسخه RouterOS روی تمام روترهای لبه
2. تطبیق با پایگاه CVE (بخش ۲۵.۲)
3. اعمال Patchهای جاافتاده (شاخه long-term upgrade)
4. ممیزی موارد نادیده‌گرفته‌شده (بخش ۲۵.۳)
5. اجرای تمرین‌های آزمایشگاهی برای تأیید عملکرد Failover پس از سخت‌سازی
6. مستندسازی یافته‌ها در Change Log
7. برنامه‌ریزی ارزیابی مجدد فصلی
```

---

**شروع ←** [تمرین‌های آزمایشگاهی](lab-exercises.md) | [مرجع CVE](cve-exploit-reference.md) | [امنیت نادیده‌گرفته‌شده](overlooked-security.md)
