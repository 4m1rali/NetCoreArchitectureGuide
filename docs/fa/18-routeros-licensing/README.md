# فصل ۱۸ — لایسنس RouterOS

> سطوح لایسنس MikroTik، الزامات Featureها و سازگاری با Multi-WAN.

---

## ۱۸.۱ نمای کلی سطوح لایسنس

MikroTik RouterOS از **سطح لایسنس نرم‌افزاری** متصل به هر دستگاه استفاده می‌کند. لایسنس تعیین می‌کند کدام قابلیت‌های پیشرفته در دسترس هستند — برای طراحی Multi-WAN حیاتی است.

| سطح | نام | کاربرد معمول | مناسب بودن برای Multi-WAN |
|-----|-----|--------------|---------------------------|
| 3 | Level 3 | SOHO، مسیریابی پایه | محدود — بدون BGP |
| 4 | Level 4 | کسب‌وکار کوچک | **حداقل برای Multi-WAN Production** |
| 5 | Level 5 | Enterprise | مجموعه کامل Featureها |
| 6 | Level 6 | ISP / Datacenter | BGP نامحدود، Full Table |

### بررسی لایسنس فعلی

```
/system license print
/system resource print
```

---

## ۱۸.۲ دسترسی Featureها بر اساس سطح لایسنس

| Feature | Level 3 | Level 4 | Level 5 | Level 6 |
|---------|---------|---------|---------|---------|
| Static routing | بله | بله | بله | بله |
| ECMP | بله | بله | بله | بله |
| PCC / Mangle | بله | بله | بله | بله |
| OSPF | خیر | بله | بله | بله |
| BGP | خیر | بله (محدود) | بله | بله (نامحدود) |
| MPLS | خیر | بله | بله | بله |
| VRF | خیر | بله | بله | بله |
| IPv6 full | بله | بله | بله | بله |
| WireGuard | بله | بله | بله | بله |
| IPsec | بله | بله | بله | بله |
| Hotspot | بله | بله | بله | بله |
| User Manager | خیر | بله | بله | بله |
| The Dude | بله | بله | بله | بله |
| Container | خیر | خیر | بله (ARM/x86) | بله |

---

## ۱۸.۳ حداقل الزامات Multi-WAN

| استقرار | حداقل لایسنس | حداقل سخت‌افزار | دلیل |
|---------|--------------|-----------------|------|
| SOHO 2-WAN Failover | Level 3 | hEX / RB750 | مسیریابی پایه کافی است |
| Enterprise 3-WAN PCC | **Level 4** | RB4011 | VRF، OSPF در صورت نیاز |
| ISP WISP 300 کاربر | **Level 5** | CCR2004+ | BGP، MPLS، تعداد Connection بالا |
| Datacenter BGP | **Level 6** | CCR2116+ | پشتیبانی Full BGP Table |
| Branch VPN + PCC | Level 4 | RB4011 | IPsec + mangle |

---

## ۱۸.۴ لایسنس و BGP

| سطح | محدودیت Routeهای BGP | Peerهای BGP |
|-----|---------------------|-------------|
| 4 | ~1000 route | 10 peer |
| 5 | ~4000 route | 50 peer |
| 6 | نامحدود | نامحدود |

برای Multi-WAN با BGP فقط Default Route (نه Full Table)، **Level 4 کافی است**.

برای Full Internet BGP Table (750,000+ route)، **Level 6 + 8GB+ RAM** الزامی است.

---

## ۱۸.۵ Trial و لایسنس CHR

### Cloud Hosted Router (CHR)

| لایسنس CHR | محدودیت سرعت | Multi-WAN |
|------------|---------------|-----------|
| Free | 1 Mbps | فقط Lab |
| P1 ($45) | 1 Gbps | استقرار کوچک |
| P10 ($95) | 10 Gbps | Enterprise |
| P-Unlimited ($250) | نامحدود | ISP/Datacenter |

CHR برای تست Lab مجازی Multi-WAN و استقرار VMware/Hyper-V ایده‌آل است.

### لایسنس Trial

سخت‌افزار جدید MikroTik شامل **Trial 60 روزه Level 6** است. از این دوره برای تست کامل Multi-WAN قبل از تثبیت لایسنس روی سطح خریداری‌شده استفاده کنید.

---

## ۱۸.۶ مسیر ارتقای لایسنس

```
Level 3 → Level 4: خرید کلید ارتقا از توزیع‌کننده MikroTik
Level 4 → Level 5: همان فرآیند
Level 5 → Level 6: همان فرآیند

/system license print
# software-id را یادداشت کنید، کلید را برای همان ID خریداری کنید
/system license renew account=your-mikrotik-account password=xxx license-key=KEY
```

---

## ۱۸.۷ تأثیر لایسنس بر تصمیمات طراحی

| اگر لایسنس... | توصیه طراحی |
|---------------|-------------|
| فقط Level 3 | فقط Failover — بدون BGP، بدون VRF |
| Level 4 | PCC + Failover — قابلیت Production کامل |
| Level 5 | افزودن BGP، MPLS، QoS پیشرفته |
| Level 6 | معماری کامل ISP با BGP Multi-Homing |

---

**فصل بعد →** [فصل ۱۹: انتخاب سخت‌افزار](../19-hardware-selection/README.md)
