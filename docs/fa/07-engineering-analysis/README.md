# فصل ۷ — تحلیل مهندسی (دیدگاه متخصص)

> تحلیل استراتژیک برای تصمیم‌گیری Multi-WAN در Production.

---

## ۷.۱ چه زمانی ECMP مناسب است — و چه زمانی خطرناک

### شرایط مناسب

| شرط | دلیل |
|-----|------|
| بدون NAT (بلوک IP عمومی Routed) | تقارن ترجمه لازم نیست |
| راه‌اندازی BGP Multi-homed | ECMP استاندارد در سطح پروتکل |
| خروجی Datacenter با بلوک /24+ عمومی | هر سرور IP عمومی خود را دارد |
| RouterOS 7 با `ecmp-per-connection=yes` | مدیریت Session بهبودیافته |
| ترافیک داخلی بین روترها | وابستگی Connection Tracking نیست |
| محیط تست و Lab | سادگی بر پایداری |

### شرایط خطرناک

| شرط | ریسک |
|-----|------|
| Masquerade NAT روی تمام WANها | **بحرانی** — Sessionها تصادفی قطع می‌شوند |
| ترافیک VoIP / SIP | سوئیچ مسیر وسط تماس باعث Drop می‌شود |
| HTTPS با Session Cookie | Sessionهای TLS با تغییر مسیر Reset می‌شوند |
| تونل‌های VPN (IPsec/OpenVPN) | مذاکره مجدد تونل با هر تغییر مسیر |
| Gaming آنلاین | NAT Type به Strict تبدیل می‌شود، Latency جهشی |
| اپلیکیشن‌های مالی / معاملاتی | تغییرات میکروثانیه‌ای مسیر باعث شکست سفارش |
| هر اپلیکیشن حساس به Session | قطعی قابل‌مشاهده برای کاربر |

### نظر متخصص

> **ECMP با NAT شایع‌ترین علت شکست Multi-WAN در استقرارهای Production میکروتیک است.** مگر IP عمومی Routed داشته باشید، از PCC استفاده کنید.

---

## ۷.۲ چرا PCC استاندارد واقعی Multi-WAN است

### توجیه فنی

| عامل | مزیت PCC |
|------|----------|
| پایداری Connection Mark | کل Session روی یک WAN — بدون سوئیچ وسط جریان |
| جداول مسیریابی به‌ازای هر WAN | کنترل مسیر مستقل به‌ازای هر ISP |
| NAT به‌ازای هر WAN | Masquerade صحیح به‌ازای هر Interface خروجی |
| سازگاری Firewall | قوانین Stateful با Connection Mark کار می‌کنند |
| Hash قطعی | توزیع قابل‌تکرار و قابل‌آزمون |
| پشتیبانی بومی MikroTik | `per-connection-classifier` داخلی — بدون ابزار خارجی |
| پذیرش صنعت ISP | الگوی استاندارد استقرار در سراسر جهان |

### چرا روش‌های دیگر استاندارد نیستند

| روش | چرا استاندارد نیست |
|-----|-------------------|
| NTH (منسوخ) | در RouterOS 6.30+ با PCC جایگزین شد |
| ECMP + NAT | شکست Session — در Production غیرقابل‌قبول |
| فقط Policy Routing | بدون توزیع خودکار — دستی به‌ازای هر Subnet |
| OSPF/BGP برای SOHO | بیش‌ازحد، نیاز به همکاری ISP |
| Bonding شخص ثالث | نیاز به تجهیزات متناظر در هر دو طرف |

### نظر متخصص

> **PCC استاندارد De Facto Multi-WAN میکروتیک است چون تنها روش داخلی است که همزمان توزیع بار، سازگاری NAT و پایداری Session را حل می‌کند.**

---

## ۷.۳ چه زمانی Failover ضروری است

### سناریوهای اجباری

| سناریو | چرا Failover لازم است |
|--------|----------------------|
| هر Multi-WAN در Production | بدون Failover، WAN مرده = ترافیک Blackhole |
| ISP با الزامات SLA | Downtime باید < ۳۰ ثانیه باشد |
| VoIP / تلفنی | تماس‌ها باید شکست WAN را تحمل کنند |
| پردازش کارت اعتباری | تداوم تراکنش لازم است |
| دفاتر شعبه دور | تکنسین حضوری برای سوئیچ دستی نیست |
| لینک‌های LTE پشتیبان | پشتیبان فقط در شکست Primary فعال می‌شود |

### فقط Failover (بدون Load Balancing)

قابل‌قبول وقتی:

- WAN ثانویه به‌طور قابل‌توجهی کندتر است (پشتیبان LTE)
- تفاوت هزینه ISPها Balancing را غیراقتصادی می‌کند
- فقط ۲ WAN و Primary ظرفیت کافی دارد
- الزام نظارتی مسیر افزون (نه پهنای باند تجمیعی)

### نظر متخصص

> **Failover در Production اختیاری نیست — همراه اجباری هر روش Load Balancing است. check-gateway را روی هر Route WAN بدون استثنا فعال کنید.**

---

## ۷.۴ بهترین ترکیب Production

### معماری پیشنهادی

```
┌─────────────────────────────────────────────┐
│           PRODUCTION STACK                   │
│                                              │
│   ┌─────────────────────────────────────┐   │
│   │  PCC (Primary Load Distribution)    │   │
│   │  • per-connection-classifier 3/0,1,2│   │
│   │  • Separate routing tables          │   │
│   │  • Per-WAN masquerade NAT           │   │
│   └─────────────────────────────────────┘   │
│                      +                       │
│   ┌─────────────────────────────────────┐   │
│   │  Failover (Gateway Monitoring)      │   │
│   │  • check-gateway=ping on all routes │   │
│   │  • Netwatch scripts for PCC disable │   │
│   │  • Main table backup routes         │   │
│   └─────────────────────────────────────┘   │
│                      +                       │
│   ┌─────────────────────────────────────┐   │
│   │  Stateful Firewall                  │   │
│   │  • established/related first        │   │
│   │  • Anti-loop WAN-to-WAN drop        │   │
│   │  • WAN input drop                   │   │
│   └─────────────────────────────────────┘   │
│                      +                       │
│   ┌─────────────────────────────────────┐   │
│   │  Policy Routing (Optional)        │   │
│   │  • VoIP → lowest latency WAN      │   │
│   │  • Servers → specific WAN         │   │
│   │  • Management → dedicated WAN     │   │
│   └─────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

### آنچه نباید ترکیب شود

| ترکیب | مشکل |
|-------|------|
| ECMP + PCC روی همان ترافیک | تصمیمات مسیریابی متضاد |
| PCC + ECMP روی همان ترافیک | طبقه‌بندی دوبل |
| Failover بدون check-gateway | Routeهای مرده فعال می‌مانند |
| PCC بدون NAT به‌ازای هر WAN | تداخل NAT در مسیر بازگشت |
| FastTrack + PCC | FastTrack Markهای Mangle را Bypass می‌کند |

---

## ۷.۵ توصیه‌های کارایی و پایداری

### اندازه‌گیری سخت‌افزار

| کاربران | مدل روتر | RAM | CPU مورد انتظار PCC |
|---------|----------|-----|---------------------|
| ۱–۵۰ | RB750Gr3 | 256MB | < 5% |
| ۵۰–۲۰۰ | RB4011 | 1GB | 5–15% |
| ۲۰۰–۱۰۰۰ | CCR2004 | 4GB | 10–25% |
| ۱۰۰۰+ | CCR2116/CCR2216 | 16GB | 15–30% |

### راهنمای بهینه‌سازی

| حوزه | توصیه |
|------|-------|
| قوانین Mangle | حداقل لازم — ۶ قانون برای PCC سه‌WAN |
| Connection Tracking | Timeoutهای TCP مناسب تنظیم کنید |
| FastTrack | در صورت استفاده از Mangle PCC غیرفعال کنید |
| DNS | از DNS Cache روتر — کاهش Queryهای WAN |
| مانیتورینگ | Netwatch هر ۱۰ ثانیه، نه هر ۱ ثانیه |
| لاگ‌گیری | Debug Firewall را در Production غیرفعال کنید |
| NTP | همیشه فعال — لاگ‌های دقیق برای عیب‌یابی |
| پشتیبان | قبل از هر تغییر Config Export کنید |

### تنظیم Connection Tracking

```
/ip firewall connection tracking
set enabled=yes tcp-established-timeout=1d tcp-time-wait-timeout=10s udp-timeout=30s
```

---

## ۷.۶ ISP در مقابل Enterprise در مقابل Datacenter

| جنبه | ISP | Enterprise | Datacenter |
|------|-----|------------|------------|
| روش اصلی | PCC + BGP | PCC + Failover | BGP + ECMP |
| تعداد WAN | ۳–۱۰+ | ۲–۳ | ۴+ |
| NAT | به‌ازای هر مشترک | Masquerade به‌ازای هر WAN | هیچ (IP عمومی) |
| Failover | Netwatch + BGP | check-gateway | BGP Failover |
| مانیتورینگ | SNMP + Netwatch | Netwatch + Syslog | BGP + SNMP |
| پیچیدگی | بسیار زیاد | زیاد | متوسط (با BGP) |
| حساسیت Session | زیاد (مشترکین) | زیاد (کاربران) | کم (Stateless) |

---

**فصل بعد ←** [فصل ۸: خلاصه نهایی معماری](../08-final-summary/README.md)
