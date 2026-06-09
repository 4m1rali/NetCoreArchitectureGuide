# Load Balancing

> چارچوب مفهومی و پیاده‌سازی عملی برای توزیع ترافیک Multi-WAN.

---

## تعریف مهندسی

**Load Balancing** در بستر Multi-WAN عمل توزیع ترافیک شبکه خروجی (و اختیاری ورودی) بین چندین لینک WAN برای حداکثرسازی استفاده از پهنای باند، کاهش ازدحام روی مسیرهای منفرد و بهبود Throughput کلی شبکه است.

Load Balancing **هدف** است — PCC و ECMP **روش‌های** دستیابی به آن هستند.

---

## مدل‌های توزیع

| مدل | لایه | دانه‌بندی | ایمن برای Session |
|-----|------|-----------|-------------------|
| Per-Packet | L3 | بسته‌های منفرد | خیر |
| Per-Flow / Per-Connection | L3/L4 | کل Connection/Session | بله |
| Per-Route | L3 | تصمیم جدول مسیریابی | بسته به شرایط |
| Weighted | L3/L4 | متناسب با ظرفیت | بله (با PCC) |

---

## جریان داخلی روتر — درخت تصمیم Load Balancing

```
Traffic arrives at edge router
    ↓
Is load balancing required?
    ├── NO → Single WAN (failover only)
    └── YES
         ↓
    Is NAT required?
         ├── YES → Use PCC (per-connection)
         └── NO
              ↓
         Is session stability required?
              ├── YES → Use PCC or ECMP per-connection
              └── NO → ECMP per-packet acceptable
                   ↓
              Are WAN capacities equal?
                   ├── YES → Equal PCC classifier (3/0, 3/1, 3/2)
                   └── NO → Weighted PCC or policy routing
```

---

## Load Balancing عملی با PCC

### توزیع مساوی (۳ WAN، ۳۳/۳۳/۳۳)

```
per-connection-classifier=both-addresses-and-ports:3/0  → WAN1
per-connection-classifier=both-addresses-and-ports:3/1  → WAN2
per-connection-classifier=both-addresses-and-ports:3/2  → WAN3
```

### توزیع وزن‌دار (۲ WAN، ۷۰/۳۰)

برای لینک‌های با ظرفیت نابرابر، از چند Bucket Classifier استفاده کنید:

```
# WAN1 (70%) — buckets 0, 1, 2, 3, 4, 5, 6
per-connection-classifier=both-addresses-and-ports:10/0
per-connection-classifier=both-addresses-and-ports:10/1
per-connection-classifier=both-addresses-and-ports:10/2
per-connection-classifier=both-addresses-and-ports:10/3
per-connection-classifier=both-addresses-and-ports:10/4
per-connection-classifier=both-addresses-and-ports:10/5
per-connection-classifier=both-addresses-and-ports:10/6

# WAN2 (30%) — buckets 7, 8, 9
per-connection-classifier=both-addresses-and-ports:10/7
per-connection-classifier=both-addresses-and-ports:10/8
per-connection-classifier=both-addresses-and-ports:10/9
```

### Load Balancing مبتنی بر سیاست

هدایت انواع خاص ترافیک به WANهای مشخص:

```
# VoIP → WAN1 (low latency)
add chain=prerouting protocol=udp dst-port=5060,5061 \
    action=mark-connection new-connection-mark=WAN1-conn

# Backup traffic → WAN3 (cheaper link)
add chain=prerouting connection-bytes=10000000-0 \
    action=mark-connection new-connection-mark=WAN3-conn
```

---

## رفتار در MikroTik RouterOS

### آنچه MikroTik می‌تواند Load Balance کند

| نوع ترافیک | روش | یادداشت |
|------------|-----|---------|
| TCP خروجی | PCC | Session Stickiness کامل |
| UDP خروجی | PCC | کار می‌کند، Timeout کوتاه‌تر |
| ICMP خروجی | ECMP یا PCC | حجم کم، معمولاً بی‌اهمیت |
| ورودی (DNAT) | Policy routing | نیاز به قوانین dst-nat به‌ازای هر WAN |
| تونل‌های VPN | Policy routing | یک تونل به‌ازای هر WAN، نه Balanced |

### آنچه MikroTik نمی‌تواند Load Balance کند

| نوع ترافیک | دلیل |
|------------|------|
| یک Connection TCP | نمی‌توان یک Connection را بین WANها تقسیم کرد |
| Connectionهای آغازشده از ورودی | Routing ISP مسیر ورودی را تعیین می‌کند |
| VPN رمزنگاری‌شده داخل تونل | تونل بیرونی یک Connection است |
| Multicast | بین مسیرهای نابرابر Routable نیست |

---

## موارد استفاده

| محیط | استراتژی |
|------|----------|
| ISP (۵۰۰+ مشترک) | PCC سه‌راهه + NAT به‌ازای هر WAN + Netwatch Failover |
| Enterprise (۱۰۰ کاربر) | PCC دو راهه مساوی + Failover |
| SOHO (۱۰ دستگاه) | PCC دو راهه یا Failover ساده |
| خروجی Datacenter | ECMP per-connection + BGP |
| شعبه با VoIP | Policy routing: VoIP→WAN1، Data→PCC |

---

## مزایا

| مزیت | جزئیات |
|------|--------|
| تجمیع پهنای باند | Throughput ترکیبی تمام لینک‌های WAN |
| کاهش ازدحام | هیچ لینکی تمام ترافیک را تحمل نمی‌کند |
| بهینه‌سازی هزینه | لینک‌های ارزان‌تر برای حجم، Premium برای Latency حساس |
| تاب‌آوری + توزیع | همراه Failover برای راه‌حل کامل |
| مقیاس‌پذیر | افزودن WAN با گسترش Bucketهای Classifier |

---

## معایب و ریسک‌ها

| ریسک | جزئیات |
|------|--------|
| Bonding واقعی نیست | 3x100Mbps ≠ 300Mbps برای یک Connection |
| سختی اندازه‌گیری | استفاده به‌ازای هر WAN بدون ابزار سخت است |
| مشکلات DNS | WANهای مختلف ممکن است به Nodeهای CDN متفاوت Resolve کنند |
| ناسازگاری Geo-IP | سرویس‌های خارجی IPهای عمومی متفاوت می‌بینند |
| مشکل Session SSL/TLS | Certificate Pinning ممکن است با تغییر IP تضاد کند |
| NAT Type در Gaming | Strict NAT روی Connectionهای Load Balanced |

---

## خطاهای رایج پیاده‌سازی

| خطا | پیامد | راه‌حل |
|-----|-------|--------|
| انتظار افزایش سرعت Single-Stream | کاربر بهبودی در دانلود نمی‌بیند | آموزش: Balancing بین Connectionهاست، نه درون یکی |
| بدون DNS به‌ازای هر WAN | DNS از WAN اشتباه Resolve می‌شود | DNS به‌ازای هر WAN یا DNS Cache روتر |
| نادیده گرفتن ظرفیت Upload | Download متعادل اما Upload شلوغ | هر دو جهت را به‌ازای هر WAN مانیتور کنید |
| Balancing بدون مانیتورینگ | WAN مرده همچنان Connection جدید دریافت می‌کند | همراه check-gateway + Netwatch |
| قوانین Mangle بیش‌ازحد | گلوگاه CPU روی روترهای پرConnection | تعداد قوانین را بهینه کنید |
| نبود پایداری Connection Mark | هر بسته مجدداً طبقه‌بندی می‌شود | passthrough=yes و جدول Connection را تأیید کنید |

---

**فصل بعد ←** [فصل ۳: جدول مقایسه](../03-comparison-table/README.md)
