# فصل ۶ — عیب‌یابی و Troubleshooting

> تحلیل خرابی‌های واقعی و ابزارهای تشخیصی MikroTik.

---

## ۶.۱ چرا اینترنت مدام قطع می‌شود

### علل ریشه‌ای

| علت | علامت | تشخیص |
|-----|-------|-------|
| Flapping Gateway | قطعی متناوب ۵–۳۰ ثانیه | `/ip route print detail` — Routeها بین active/inactive جابه‌جا می‌شوند |
| مثبت کاذب check-gateway | قطعی در زمان ازدحام | Timeout Ping روی لینک شلوغ |
| شکست DNS | «متصل» اما صفحات باز نمی‌شوند | `/ping 8.8.8.8` کار می‌کند، `/ping google.com` شکست می‌خورد |
| پر شدن جدول Connection | افت تدریجی کارایی | `/ip firewall connection print count-only` |
| مشکل سمت ISP | Gateway قابل‌دسترس، بدون اینترنت | Ping Gateway OK، ping 8.8.8.8 شکست |
| پر شدن جدول NAT | Connectionهای جدید شکست می‌خورند | `/ip firewall nat print` — تعداد ورودی بالا |

### دستورات تشخیصی

```
/ip route print detail where inactive
/ip route print detail where check-gateway
/ping 203.0.113.1 count=50
/ping 8.8.8.8 count=50
/tool netwatch print
/log print where topics~"route"
```

### مراحل رفع مشکل

1. وضعیت Route را بررسی کنید: `/ip route print detail`
2. دسترسی Gateway را تأیید کنید: `/ping <gateway-ip> count=20`
3. دسترسی اینترنت را تأیید کنید: `/ping 8.8.8.8 count=20`
4. Flapping Route در لاگ‌ها: `/log print`
5. در صورت مثبت کاذب، Timeout check-gateway را افزایش دهید
6. Netwatch برای مانیتورینگ Upstream (8.8.8.8) اضافه کنید

---

## ۶.۲ چرا Load Balancing ناپایدار می‌شود

### علل ریشه‌ای

| علت | علامت | تشخیص |
|-----|-------|-------|
| نبود passthrough=yes | WAN تصادفی به‌ازای هر بسته | `/ip firewall mangle print` — passthrough را بررسی کنید |
| ترتیب نادرست قوانین Mangle | توزیع نابرابر | `/ip firewall mangle print stats` |
| FastTrack که Mangle را Bypass می‌کند | برخی Connectionها بدون Mark | `/ip firewall filter print` — قوانین fasttrack |
| Connection Mark پایدار نمی‌ماند | Sessionها وسط جریان WAN عوض می‌کنند | `/ip firewall connection print` — Markها را بررسی کنید |
| یک Route WAN Inactive | ۵۰٪ ترافیک به‌جای ۳۳٪ | `/ip route print` — Routeهای inactive |
| مسیریابی نامتقارن | دانلود کار می‌کند، آپلود شکست | `/tool torch` — هر دو جهت را بررسی کنید |

### دستورات تشخیصی

```
/ip firewall mangle print stats
/ip firewall connection print where connection-mark!=""
/ip route print where routing-table~"to-WAN"
/tool torch interface=ether5 duration=30
/interface monitor-traffic ether1,ether2,ether3 once
```

### تأیید توزیع

```
/ip firewall connection print count-only where connection-mark="WAN1-conn"
/ip firewall connection print count-only where connection-mark="WAN2-conn"
/ip firewall connection print count-only where connection-mark="WAN3-conn"
```

انتظار: تعداد تقریباً مساوی در هر سه Mark WAN.

### مراحل رفع مشکل

1. ترتیب قوانین Mangle و تنظیم passthrough را تأیید کنید
2. در صورت فعال بودن FastTrack غیرفعال کنید (با Mangle PCC تضاد دارد)
3. تأیید کنید تمام Routeهای جدول PCC فعال هستند
4. توزیع Connection Mark را بررسی کنید (باید ~33/33/33 باشد)
5. ترافیک به‌ازای هر Interface را برای تأیید تعادل مانیتور کنید

---

## ۶.۳ چرا مسیریابی اشتباه رخ می‌دهد

### علل ریشه‌ای

| علت | علامت | تشخیص |
|-----|-------|-------|
| نبود جدول مسیریابی | ترافیک از جدول main استفاده می‌کند | `/routing table print` |
| Routing Mark تنظیم نشده | PCC نادیده گرفته می‌شود | `/ip firewall mangle print stats` |
| Route Recursive نبود | Gateway غیرقابل‌دسترس | `/ip route print detail where dst-address~"GW-IP"` |
| scope/target-scope نادرست | Route برای Forwarding استفاده نمی‌شود | `/ip route print detail` — scope را بررسی کنید |
| تضاد Distance | Route Failover PCC را Override می‌کند | مقایسه Routeهای main و PCC |
| پیکربندی نادرست VRF | ترافیک در FIB اشتباه | `/routing table print` |

### دستورات تشخیصی

```
/ip route print detail
/routing table print
/ip route print where routing-mark!=""
/ip firewall mangle print stats
/routing rule print
```

### Trace تصمیم Route

```
/tool sniffer quick interface=ether5 ip-protocol=tcp port=443
/tool traceroute 8.8.8.8
```

### مراحل رفع مشکل

1. وجود جداول مسیریابی را تأیید کنید: `/routing table print`
2. Routeهای PCC در جداول صحیح هستند (نه main)
3. Routeهای Recursive Gateway دارای scope=10 هستند
4. قوانین mangle routing-mark ترافیک دارند
5. Route متضاد در جدول main با Distance کمتر وجود ندارد

---

## ۶.۴ چرا تداخل NAT رخ می‌دهد

### علل ریشه‌ای

| علت | علامت | تشخیص |
|-----|-------|-------|
| یک Masquerade برای تمام WANها | ترافیک بازگشتی از WAN اشتباه | `/ip firewall nat print` |
| NAT قبل از Routing Mark | out-interface اشتباه انتخاب می‌شود | ترتیب قوانین در زنجیره nat |
| قوانین NAT تکراری | ترجمه دوبل | `/ip firewall nat print` — ورودی‌های تکراری |
| برخورد Port | شکست تصادفی Connection | دو WAN همزمان همان Port را NAT می‌کنند |
| نبود Hairpin NAT | دسترسی داخلی به IP عمومی شکست | بدون قانون dstnat loopback |
| عدم تطابق Connection Mark | NAT اعمال شد اما IP WAN اشتباه | تطبیق conn mark و Interface NAT |

### دستورات تشخیصی

```
/ip firewall nat print
/ip firewall connection print where src-address~"192.168"
/tool sniffer quick interface=ether2 ip-address=198.51.100.2
/ip firewall nat print stats
```

### مراحل رفع مشکل

1. قوانین Masquerade جداگانه به‌ازای هر out-interface
2. out-interface NAT با مسیر PCC تطابق دارد
3. قوانین NAT تکراری یا متضاد وجود ندارد
4. Connection Tracking آدرس‌های ترجمه‌شده صحیح را نشان می‌دهد
5. در صورت استفاده سرورهای داخلی از DNS عمومی، Hairpin NAT اضافه کنید

---

## ۶.۵ جعبه ابزار Debug میکروتیک

### دستورات Debug ضروری

| ابزار | دستور | هدف |
|-------|-------|-----|
| وضعیت Route | `/ip route print detail` | Routeهای active/inactive، Gatewayها |
| جدول Connection | `/ip firewall connection print` | Sessionهای فعال، Markها، NAT |
| آمار Mangle | `/ip firewall mangle print stats` | شمارنده برخورد قوانین |
| آمار NAT | `/ip firewall nat print stats` | شمارنده برخورد قوانین NAT |
| مانیتور ترافیک | `/interface monitor-traffic <iface>` | پهنای باند لحظه‌ای به‌ازای هر WAN |
| Torch | `/tool torch <iface>` | تحلیل‌گر Connection زنده |
| Sniffer | `/tool sniffer quick <iface>` | ضبط بسته |
| Traceroute | `/tool traceroute <ip>` | تأیید مسیر |
| Ping | `/ping <ip> count=50` | Latency و Packet Loss |
| تست پهنای باند | `/tool bandwidth-test` | اندازه‌گیری Throughput WAN |
| Netwatch | `/tool netwatch print` | وضعیت مانیتور Gateway |
| لاگ‌ها | `/log print` | رویدادها و خطاهای سیستم |

### گردش کار Debug

```
STEP 1: Verify physical layer
  /interface print stats
  /interface ethernet print

STEP 2: Verify IP and gateway
  /ip address print
  /ip route print detail

STEP 3: Verify PCC classification
  /ip firewall mangle print stats
  /ip firewall connection print count-only

STEP 4: Verify NAT
  /ip firewall nat print stats
  /ip firewall connection print where protocol=tcp

STEP 5: Verify traffic flow
  /interface monitor-traffic ether1,ether2,ether3
  /tool torch interface=ether5

STEP 6: Check logs
  /log print where topics~"firewall,route,system"
```

### Packet Sniffer برای Debug PCC

```
/tool sniffer
set filter-interface=ether5 filter-ip-protocol=tcp \
    filter-connection-mark=WAN2-conn streaming-enabled=yes

/tool sniffer quick interface=ether5 duration=10
```

---

## ۶.۶ مرجع سریع مشکل → راه‌حل

| مشکل | راه‌حل سریع |
|------|-------------|
| اصلاً اینترنت نیست | Default Route فعال، ping Gateway |
| اینترنت فقط از یک WAN | قوانین Mangle PCC نبود یا غیرفعال |
| قطعی متناوب | check-gateway بیش‌ازحد تهاجمی، Timeout را افزایش دهید |
| مرور کند | مشکل DNS — `/ip dns print` را بررسی کنید |
| VPN کار نمی‌کند | Policy Route VPN به یک WAN |
| VoIP قطع‌قطع | Policy Route VoIP به WAN با کمترین Latency |
| آپلود کار می‌کند، دانلود نه | تداخل NAT — Masquerade به‌ازای هر WAN |
| CPU بالا | قوانین Mangle زیاد یا جدول Connection پر |
| پس از Reboot بدون Balancing | ترتیب Startup را تأیید کنید، جداول مسیریابی پایدار |
| یک WAN همیشه ۰٪ | Route Inactive — check-gateway شکست می‌خورد |

---

**فصل بعد ←** [فصل ۷: تحلیل مهندسی](../07-engineering-analysis/README.md)
