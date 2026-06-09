# فصل ۸ — خلاصه نهایی معماری

> نتیجه‌گیری‌های مهندسی و توصیه‌های استقرار Production.

---

## ۸.۱ بهترین طراحی Multi-WAN

### معماری مرجع

```
                    ┌──────────────────────────────┐
                    │     PRODUCTION MULTI-WAN      │
                    │                              │
   ISP-1 ──────────│  PCC Bucket 0  → to-WAN1    │
   (Primary)       │  ISP-2 ────────│  PCC Bucket 1  → to-WAN2    │
                   │  ISP-3 ────────│  PCC Bucket 2  → to-WAN3    │
   (Backup)        │                              │
                   │  Failover: check-gateway     │
                   │  NAT: per-WAN masquerade     │
                   │  Firewall: stateful + anti-  │
                   │            loop              │
                   │  Monitor: Netwatch + logs    │
                    └──────────────────────────────┘
                              │
                         LAN 192.168.1.0/24
```

### اصول طراحی

| # | اصل | پیاده‌سازی |
|---|-----|-----------|
| ۱ | Session Stickiness | Connection Markهای PCC |
| ۲ | افزونگی مسیر | check-gateway + Netwatch |
| ۳ | تقارن NAT | Masquerade به‌ازای هر Interface |
| ۴ | امنیت Default-Deny | Stateful Firewall، Drop ورودی WAN |
| ۵ | قابل‌مشاهده | آمار Mangle، مانیتور ترافیک، لاگ‌ها |
| ۶ | قابل‌بازیابی | Export Config، طرح IP مستند |

---

## ۸.۲ بهترین روش توزیع بار

| رتبه | روش | امتیاز | نظر |
|------|-----|--------|-----|
| ۱ | **PCC** | ★★★★★ | **استاندارد Production** — از این استفاده کنید |
| ۲ | ECMP per-connection | ★★★☆☆ | قابل‌قبول فقط بدون NAT |
| ۳ | ECMP per-packet | ★★☆☆☆ | فقط Lab/تست |
| ۴ | Policy routing | ★★★☆☆ | مکمل PCC، نه جایگزین |
| ۵ | NTH (قدیمی) | ★☆☆☆☆ | منسوخ — استفاده نکنید |

### استراتژی توزیع بر اساس تعداد WAN

| WANها | Classifier | تقسیم |
|-------|-----------|-------|
| ۲ | `:2/0`، `:2/1` | ۵۰٪ / ۵۰٪ |
| ۳ | `:3/0`، `:3/1`، `:3/2` | ۳۳٪ / ۳۳٪ / ۳۳٪ |
| ۴ | `:4/0` تا `:4/3` | ۲۵٪ هر کدام |
| ۲ (۷۰/۳۰) | `:10/0-6`، `:10/7-9` | ۷۰٪ / ۳۰٪ |

---

## ۸.۳ بهترین ترکیب Failover + PCC + ECMP

### فرمول Production

```
PCC (traffic distribution)
  + Failover (gateway monitoring)
  + Per-WAN NAT (address translation)
  + Stateful Firewall (security)
  + Netwatch (advanced monitoring)
  = Production-Ready Multi-WAN
```

### مسئولیت هر جزء

| جزء | مسئولیت | زمان فعال‌سازی |
|-----|---------|----------------|
| PCC | توزیع Connectionهای جدید بین WANها | هر Connection جدید |
| Failover | سوئیچ به پشتیبان هنگام مرگ Gateway | شکست Ping Gateway |
| NAT به‌ازای هر WAN | IP عمومی صحیح به‌ازای هر مسیر خروجی | هر بسته ترجمه‌شده |
| Firewall | مسدودسازی ترافیک غیرمجاز | هر بسته |
| Netwatch | غیرفعال‌سازی Route PCC برای WAN مرده | شکست مانیتور Gateway |
| ECMP | **استفاده نمی‌شود** در محیط‌های NAT | N/A |

### نقش ECMP در این معماری

ECMP **بخشی از ترکیب Production نیست** وقتی NAT درگیر است. ECMP همچنان در دسترس است به‌عنوان:

- جایگزین برای استقرار IP عمومی Routed
- سناریوهای BGP Datacenter
- مسیرهای داخلی روتر به روتر

---

## ۸.۴ توصیه‌های کارایی

| حوزه | توصیه | تأثیر |
|------|-------|-------|
| غیرفعال‌سازی FastTrack | جلوگیری از Bypass Mangle | بحرانی برای PCC |
| DNS Cache روتر | کاهش ترافیک DNS WAN | متوسط |
| تنظیم Timeout TCP | جلوگیری از انباشت جدول Connection | زیاد در مقیاس |
| روتر سخت‌افزاری (CCR) | Mangle در Line Rate | بحرانی > ۲۰۰ کاربر |
| قوانین Mangle حداقلی | ۶ قانون برای ۳-WAN — نه بیشتر | صرفه‌جویی CPU |
| Interval Netwatch 10s | تعادل سرعت تشخیص و مثبت کاذب | پایداری |
| برنامه Export Config | پشتیبان خودکار هفتگی | بازیابی |

---

## ۸.۵ توصیه‌های پایداری

| حوزه | توصیه | تأثیر |
|------|-------|-------|
| check-gateway روی تمام Routeها | بدون ترافیک Blackhole | بحرانی |
| Drop Anti-loop WAN-to-WAN | جلوگیری از Loop مسیریابی | بحرانی |
| NTP فعال | لاگ‌های عیب‌یابی دقیق | زیاد |
| غیرفعال‌سازی سرویس‌های بلااستفاده | کاهش سطح حمله | زیاد |
| rp-filter=strict | جلوگیری از IP Spoofing | متوسط |
| مانیتور IP خارجی (8.8.8.8) | تشخیص شکست سمت ISP | زیاد |
| مستندسازی طرح IP | عیب‌یابی سریع‌تر | زیاد |
| تست Failover ماهانه | تأیید مسیر پشتیبان | بحرانی |

---

## ۸.۶ چک‌لیست استقرار

| # | وظیفه | وضعیت |
|---|-------|-------|
| ۱ | لیست Interface (WAN/LAN) پیکربندی شد | ☐ |
| ۲ | آدرس‌های IP به‌ازای هر WAN و LAN تخصیص یافت | ☐ |
| ۳ | Routeهای Recursive Gateway ایجاد شد | ☐ |
| ۴ | جداول مسیریابی ایجاد شد (to-WAN1/2/3) | ☐ |
| ۵ | Routeهای PCC در جداول به‌ازای هر WAN با check-gateway | ☐ |
| ۶ | Routeهای Failover در جدول main با اولویت Distance | ☐ |
| ۷ | قوانین Mangle PCC با passthrough=yes | ☐ |
| ۸ | قوانین Mangle Routing Mark پیکربندی شد | ☐ |
| ۹ | قوانین Masquerade NAT به‌ازای هر WAN | ☐ |
| ۱۰ | قوانین Stateful Firewall (established اول) | ☐ |
| ۱۱ | قوانین Anti-loop و Drop ورودی WAN | ☐ |
| ۱۲ | اسکریپت‌های Netwatch برای مانیتورینگ Gateway | ☐ |
| ۱۳ | DNS و DHCP پیکربندی شد | ☐ |
| ۱۴ | سرویس‌های بلااستفاده غیرفعال شد | ☐ |
| ۱۵ | NTP فعال شد | ☐ |
| ۱۶ | Config Export و پشتیبان شد | ☐ |
| ۱۷ | Failover تست شد (قطع یک WAN) | ☐ |
| ۱۸ | توزیع بار تأیید شد (آمار mangle) | ☐ |
| ۱۹ | NAT تأیید شد (بررسی جدول connection) | ☐ |
| ۲۰ | Baseline کارایی ثبت شد | ☐ |

---

## ۸.۷ نتیجه‌گیری نهایی مهندسی

> **برای استقرارهای Production Multi-WAN میکروتیک با NAT، معماری قطعی این است:**
>
> **PCC** برای توزیع بار به‌ازای هر Connection بین ۲–۳ لینک ISP، با **Failover check-gateway** روی هر Route، **Masquerade NAT به‌ازای هر WAN** برای تقارن Session، **Stateful Firewall** برای امنیت، و **Netwatch** برای مانیتورینگ پیشگیرانه Gateway.
>
> این ترکیب Load Balancing پایدار Session، Failover خودکار، رفتار NAT صحیح و امنیت در سطح Production را ارائه می‌دهد — همان معماری که ISPها و سازمان‌ها در سراسر جهان روی MikroTik RouterOS استقرار داده‌اند.

---

## ۸.۸ ادامه یادگیری — فصل‌های پیشرفته

| موضوع | فصل |
|-------|-----|
| Hairpin NAT، DNS، Port Forwarding | [فصل ۹](../09-advanced-nat-dns/README.md) |
| QoS، مدیریت پهنای باند | [فصل ۱۰](../10-qos-traffic/README.md) |
| VPN روی Multi-WAN | [فصل ۱۱](../11-vpn-multiwan/README.md) |
| IPv6 Dual-Stack | [فصل ۱۲](../12-ipv6-multiwan/README.md) |
| BGP Multi-Homing | [فصل ۱۳](../13-bgp-multihoming/README.md) |
| مانیتورینگ و NOC | [فصل ۱۴](../14-monitoring-operations/README.md) |
| PPPoE، LTE، DHCP WAN | [فصل ۱۵](../15-wan-types/README.md) |
| سخت‌سازی امنیتی | [فصل ۱۶](../16-security-hardening/README.md) |
| مطالعات موردی واقعی | [فصل ۱۷](../17-case-studies/README.md) |
| مرجع سریع | [Cheat Sheet](../appendix/cheat-sheet.md) |

---

## ۸.۹ فصل‌های مرجع (۱۸–۲۴)

| موضوع | فصل |
|-------|-----|
| سطوح لایسنس RouterOS | [فصل ۱۸](../18-routeros-licensing/README.md) |
| Sizing سخت‌افزار | [فصل ۱۹](../19-hardware-selection/README.md) |
| Migration ROS 6 → 7 | [فصل ۲۰](../20-migration-upgrade/README.md) |
| VRRP / HA | [فصل ۲۱](../21-high-availability/README.md) |
| Hotspot + PCC | [فصل ۲۲](../22-hotspot-captive-portal/README.md) |
| Scriptهای Automation | [فصل ۲۳](../23-automation-scripting/README.md) |
| Disaster Recovery | [فصل ۲۴](../24-disaster-recovery/README.md) |
| FAQ | [پیوست ج](../appendix/faq.md) |
| اشتباهات رایج | [پیوست د](../appendix/common-mistakes.md) |
| آزمایشگاه امنیت و CVEها | [فصل ۲۵](../25-security-labs-cve/README.md) |

---

**[← بازگشت به فهرست اصلی](../README.md)** | **[لایسنس →](../../LICENSE)**
