# فصل ۳ — جدول مقایسه Enterprise

> ماتریس حرفه‌ای مقایسه روش‌ها برای تصمیم‌گیری استقرار Multi-WAN.

---

## ماتریس مقایسه روش‌های Multi-WAN

| ویژگی | ECMP | PCC | Failover | ECMP + Failover | PCC + Failover |
|-------|------|-----|----------|-----------------|----------------|
| **نوع روش** | Routing | Mangle + Routing | Route Priority | ترکیبی | ترکیبی |
| **لایه OSI** | L3 | L3/L4 | L3 | L3 | L3/L4 |
| **مدل توزیع بار** | Packet / Flow | Flow (Per-Connection) | None (Active/Standby) | Flow | Flow |
| **پایداری Session** | ضعیف (per-packet) / خوب (per-conn) | عالی | N/A (مسیر واحد) | خوب | عالی |
| **سازگاری NAT** | ضعیف | عالی | خوب | متوسط | عالی |
| **تأثیر کارایی** | حداقلی | کم–متوسط | حداقلی | کم | کم–متوسط |
| **مصرف CPU** | بسیار کم | متوسط | بسیار کم | کم | متوسط |
| **مقیاس‌پذیری** | بالا (افزودن Route) | بالا (افزودن Bucket) | بالا | بالا | بالا |
| **سطح پیچیدگی** | کم | زیاد | کم | متوسط | زیاد |
| **ریسک خرابی** | متوسط (شکست NAT) | کم | بسیار کم | کم | بسیار کم |
| **بهترین کاربرد** | IP عمومی Routed، بدون NAT | محیط‌های NAT، ISP/Enterprise | WAN پشتیبان، Uptime حیاتی | Datacenter، BGP | **Multi-WAN Production (توصیه‌شده)** |

---

## تحلیل تفصیلی ویژگی‌ها

### مدل توزیع بار

| روش | مدل | توضیح |
|-----|-----|-------|
| ECMP (پیش‌فرض) | Per-Packet | هر بسته جداگانه Hash — سریع‌ترین اما NAT را می‌شکند |
| ECMP (per-conn) | Per-Flow | هر Connection یک‌بار Hash — سازگاری بهتر با NAT |
| PCC | Per-Flow | Connection Mark برای تمام عمر Session پایدار می‌ماند |
| Failover | Active/Standby | تمام ترافیک روی Primary تا زمان خرابی |

### امتیاز پایداری Session

| روش | امتیاز | جزئیات |
|-----|--------|--------|
| ECMP per-packet | ★☆☆☆☆ | Sessionها با NAT مرتباً قطع می‌شوند |
| ECMP per-connection | ★★★☆☆ | پایدار درون Connection، بدون کنترل NAT به‌ازای هر WAN |
| PCC | ★★★★★ | Session Stickiness کامل با NAT به‌ازای هر WAN |
| Failover | ★★★★☆ | پایدار تا رویداد Failover (یک‌بار Drop Session) |

### امتیاز سازگاری NAT

| روش | امتیاز | جزئیات |
|-----|--------|--------|
| ECMP per-packet | ★☆☆☆☆ | Masquerade مسیر بازگشت را می‌شکند |
| ECMP per-connection | ★★★☆☆ | با یک قانون Masquerade کار می‌کند |
| PCC | ★★★★★ | Masquerade به‌ازای هر Interface با Connection Mark |
| Failover | ★★★★☆ | یک قانون NAT فعال کافی است |

### مقایسه مصرف CPU

| روش | هزینه Connection جدید | هزینه Per-Packet | ۱۰K Connection |
|-----|----------------------|------------------|----------------|
| ECMP | هیچ | فقط Hash | ~0% CPU |
| PCC | ارزیابی Mangle | Lookup Mark | ~5–15% CPU |
| Failover | فقط Ping | هیچ | ~1% CPU |
| PCC + Failover | Mangle + ping | Lookup Mark | ~5–20% CPU |

### سطح پیچیدگی

| روش | آیتم‌های Config | جداول مسیریابی | قوانین Mangle | قوانین NAT |
|-----|----------------|----------------|---------------|------------|
| ECMP | ۳–۵ | ۱ (main) | ۰ | ۱ |
| PCC | ۱۵–۲۵ | ۴ (main + ۳ WAN) | ۶–۹ | ۳ |
| Failover | ۳–۵ | ۱ (main) | ۰ | ۱ |
| PCC + Failover | ۲۰–۳۰ | ۴ | ۶–۹ | ۳ |

### ارزیابی ریسک خرابی

| روش | سطح ریسک | ریسک اصلی |
|-----|----------|-----------|
| ECMP + NAT | **بالا** | شکست Session، مسیریابی نامتقارن |
| PCC | **کم** | ترتیب نادرست Mangle |
| فقط Failover | **بسیار کم** | نقطه واحد پهنای باند |
| PCC + Failover | **بسیار کم** | Connection یتیم PCC روی WAN شکست‌خورده (کاهش با Netwatch) |

---

## ماتریس تصمیم

| نیاز شما | روش پیشنهادی |
|----------|--------------|
| NAT + Load Balancing + Failover | **PCC + Failover** |
| مسیریابی IP عمومی، بدون NAT | **ECMP + Failover** |
| حداکثر Uptime، یک WAN کافی است | **فقط Failover** |
| Lab / تست ساده | **ECMP** |
| ISP با ۳ Upstream | **PCC + Failover + NAT به‌ازای هر WAN** |
| Enterprise با ۱۰۰+ کاربر | **PCC + Failover** |
| ترافیک ترکیبی VoIP + Data | **PCC + Policy Routing + Failover** |
| Datacenter BGP Multi-homed | **BGP + ECMP** |

---

## ماتریس پشتیبانی قابلیت (MikroTik RouterOS)

| قابلیت | ECMP | PCC | Failover |
|--------|------|-----|----------|
| RouterOS 6.x | بله | بله | بله |
| RouterOS 7.x | بله (بهبودیافته) | بله | بله |
| IPv6 | بله | بله | بله |
| VRF | بله | بله | بله |
| سازگار با FastTrack | بله | جزئی (Mangle را Bypass می‌کند) | بله |
| Hardware offload | بله | خیر (Mangle) | بله |
| اینترفیس VLAN | بله | بله | بله |
| WAN PPPoE | بله | بله | بله |
| WAN Bridge | توصیه نمی‌شود | توصیه نمی‌شود | بله |

---

**فصل بعد ←** [فصل ۴: طراحی Production](../04-production-design/README.md)
