# ECMP Routing

> Equal-Cost Multi-Path — توزیع بار در سطح مسیریابی L3.

---

## تعریف مهندسی

**ECMP (Equal-Cost Multi-Path)** مکانیزم مسیریابی است که در آن چندین مسیر Next-Hop به یک Prefix مقصد دارای Metric (Distance) یکسان هستند و روتر ترافیک را به‌طور همزمان بین تمام مسیرهای موجود توزیع می‌کند.

در سطح Enterprise، ECMP صرفاً در **لایه ۳** عمل می‌کند — هر بسته (یا Flow، بسته به پیاده‌سازی) را بدون آگاهی Session به یکی از Gatewayهای Equal-Cost هش می‌کند.

---

## جریان داخلی روتر (گام‌به‌گام)

```
1. Packet arrives with destination 0.0.0.0/0 (default route)
2. Routing table lookup in "main" table
3. Multiple routes found with SAME distance:
   → 0.0.0.0/0 via 203.0.113.1 distance=1
   → 0.0.0.0/0 via 198.51.100.1 distance=1
   → 0.0.0.0/0 via 192.0.2.1 distance=1
4. ECMP hash algorithm applied:
   → Input: src-IP, dst-IP, protocol, src-port, dst-port
   → Output: selected gateway index (0, 1, or 2)
5. Packet forwarded via selected gateway interface
6. NEXT packet in same TCP session MAY use different gateway
   (no connection tracking involvement in pure ECMP)
```

---

## رفتار در MikroTik RouterOS

### الگوی پیکربندی

```
/ip route
add dst-address=0.0.0.0/0 gateway=203.0.113.1 distance=1 check-gateway=ping
add dst-address=0.0.0.0/0 gateway=198.51.100.1 distance=1 check-gateway=ping
add dst-address=0.0.0.0/0 gateway=192.0.2.1 distance=1 check-gateway=ping
```

### رفتار اختصاصی MikroTik

| رفتار | جزئیات |
|-------|--------|
| ورودی Hash | فیلدهای هدر لایه ۳/۴ |
| مدل توزیع | Per-packet (پیش‌فرض) یا Per-connection (با ECMP per-connection-setting) |
| حذف مسیر | با شکست check-gateway، مسیر Inactive می‌شود — ترافیک بازتوزیع می‌شود |
| Scope جدول | فقط در یک جدول مسیریابی عمل می‌کند |
| تعامل با NAT | مشکل‌ساز — Sessionها ممکن است به‌خاطر تغییر مسیر Per-packet قطع شوند |

### حالت ECMP Per-Connection (RouterOS 7+)

```
/routing settings
set ecmp-per-connection=yes
```

این تنظیم ECMP را شبیه توزیع مبتنی بر Flow می‌کند و سازگاری با NAT را بهبود می‌دهد.

---

## موارد استفاده

| محیط | کاربرد |
|------|--------|
| Core ISP | چندین لینک Peer Upstream با BGP |
| Datacenter | مسیرهای Equal-Cost به Internet Exchange |
| Enterprise (بدون NAT) | بلوک‌های IP عمومی Routed بین WANها |
| Lab/Testing | تجمیع ساده پهنای باند بدون پیچیدگی PCC |

---

## مزایا

| مزیت | جزئیات |
|------|--------|
| سادگی | پیکربندی حداقلی — فقط افزودن Route با Distance یکسان |
| کارایی | بدون قانون Mangle، بدون Connection Mark — کمترین Overhead CPU |
| مقیاس‌پذیری | افزودن WAN با اضافه کردن Route |
| همگرایی سریع | check-gateway مسیرهای مرده را سریع حذف می‌کند |
| مسیریابی بومی | از جدول مسیریابی استاندارد — بدون جدول سفارشی |

---

## معایب و ریسک‌ها

| ریسک | جزئیات |
|------|--------|
| شکست NAT | هش Per-packet تقارن Connection Tracking را می‌شکند |
| ناپایداری Session | Sessionهای HTTPS، VPN، VoIP ممکن است وسط Connection Reset شوند |
| بدون سیاست ترافیک | نمی‌توان ترافیک خاص را به WAN مشخص هدایت کرد |
| مسیریابی نامتقارن | مسیر بازگشت ممکن است متفاوت باشد اگر ISPها Routing متفاوت داشته باشند |
| بدون وزن‌دهی پهنای باند | تمام مسیرها یکسان در نظر گرفته می‌شوند صرف‌نظر از ظرفیت |
| مشکل Stateful Firewall | `connection-state=established` ممکن است با تغییر مسیر شکست بخورد |

---

## خطاهای رایج پیاده‌سازی

| خطا | پیامد | راه‌حل |
|-----|-------|--------|
| ECMP با Masquerade NAT | Sessionهای شکسته، اتصال متناوب | از PCC استفاده کنید |
| نبود check-gateway | مسیرهای مرده فعال می‌مانند، ترافیک Blackhole | `check-gateway=ping` به تمام Routeها اضافه کنید |
| Distance متفاوت | فقط یک Route استفاده می‌شود، بدون Load Balancing | Distance یکسان برای تمام Routeها |
| ECMP + PCC همزمان | تصمیمات مسیریابی متضاد | یک روش به‌ازای هر کلاس ترافیک انتخاب کنید |
| بدون قوانین Stateful Firewall | وضعیت Connection نامعتبر | ابتدا قوانین accept established/related اضافه کنید |

---

**بعدی ←** [PCC (Per Connection Classifier)](pcc.md)
