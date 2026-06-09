# PCC (Per Connection Classifier)

> روش استاندارد صنعتی برای Load Balancing چند-WAN روی MikroTik.

---

## تعریف مهندسی

**PCC (Per Connection Classifier)** تکنیک طبقه‌بندی ترافیک است که از Hash قطعی پارامترهای Connection (Source IP، Destination IP، Source Port، Destination Port، Protocol) برای تخصیص هر Connection جدید به یک مسیر WAN مشخص استفاده می‌کند. پس از طبقه‌بندی، Connection برای تمام عمر خود از طریق Connection Mark مسیر تخصیص‌یافته را حفظ می‌کند.

PCC در **لایه ۳/۴** از طریق Mangle Firewall و سیستم Routing-Mark میکروتیک عمل می‌کند.

---

## جریان داخلی روتر (گام‌به‌گام)

```
1. NEW packet arrives (no connection table entry)
2. Mangle PREROUTING chain:
   a. Check: connection-mark is empty (new connection)
   b. Calculate PCC hash:
      hash = (src-ip XOR dst-ip XOR src-port XOR dst-port) MOD N
      where N = number of WAN links
   c. Result 0 → assign connection-mark "WAN1-conn"
      Result 1 → assign connection-mark "WAN2-conn"
      Result 2 → assign connection-mark "WAN3-conn"
   d. Set routing-mark based on connection-mark
3. Connection tracking: STORE connection-mark
4. Routing: use routing table matching routing-mark
   → "to-WAN1" / "to-WAN2" / "to-WAN3"
5. NAT: masquerade on correct out-interface
6. SUBSEQUENT packets (ESTABLISHED):
   a. Connection table provides stored connection-mark
   b. Mangle rules SKIPPED (mark already set)
   c. Same routing table, same NAT, same WAN path
7. Connection closes → entry removed from table
```

---

## رفتار در MikroTik RouterOS

### الگوی قانون Mangle PCC

```
/ip firewall mangle
add chain=prerouting dst-address-type=!local in-interface-list=LAN \
    per-connection-classifier=both-addresses-and-ports:3/0 \
    action=mark-connection new-connection-mark=WAN1-conn passthrough=yes

add chain=prerouting dst-address-type=!local in-interface-list=LAN \
    per-connection-classifier=both-addresses-and-ports:3/1 \
    action=mark-connection new-connection-mark=WAN2-conn passthrough=yes

add chain=prerouting dst-address-type=!local in-interface-list=LAN \
    per-connection-classifier=both-addresses-and-ports:3/2 \
    action=mark-connection new-connection-mark=WAN3-conn passthrough=yes
```

### سینتکس Classifier

```
per-connection-classifier=both-addresses-and-ports:N/M
```

| پارامتر | معنی |
|---------|------|
| `both-addresses-and-ports` | ورودی Hash: src-ip + dst-ip + src-port + dst-port |
| `N` | تعداد کل لینک‌های WAN |
| `M` | شاخص Bucket (۰ تا N-1) |

### تخصیص Routing Mark

```
add chain=prerouting connection-mark=WAN1-conn in-interface-list=LAN \
    action=mark-routing new-routing-mark=to-WAN1 passthrough=yes

add chain=prerouting connection-mark=WAN2-conn in-interface-list=LAN \
    action=mark-routing new-routing-mark=to-WAN2 passthrough=yes

add chain=prerouting connection-mark=WAN3-conn in-interface-list=LAN \
    action=mark-routing new-routing-mark=to-WAN3 passthrough=yes
```

### جداول مسیریابی

```
/ip route
add dst-address=0.0.0.0/0 gateway=203.0.113.1 routing-table=to-WAN1 check-gateway=ping
add dst-address=0.0.0.0/0 gateway=198.51.100.1 routing-table=to-WAN2 check-gateway=ping
add dst-address=0.0.0.0/0 gateway=192.0.2.1 routing-table=to-WAN3 check-gateway=ping
```

---

## موارد استفاده

| محیط | چرا PCC |
|------|---------|
| روتر لبه ISP | توزیع ترافیک مشترک بین Upstreamها |
| شعبه Enterprise | Load Balancing پایدار Session برای ۵۰–۵۰۰ کاربر |
| SOHO با ۲ ISP | Dual-WAN قابل‌اعتماد بدون قطع Session |
| VoIP + Data ترکیبی | هر تماس روی یک مسیر می‌ماند (بدون سوئیچ وسط تماس) |
| Gaming / Streaming | Latency پایدار به‌ازای هر Session |

---

## مزایا

| مزیت | جزئیات |
|------|--------|
| پایداری Session | Connection Mark برای تمام عمر Session پایدار می‌ماند |
| سازگار با NAT | با masquerade و src-nat درست کار می‌کند |
| توزیع قابل‌پیش‌بینی | الگوریتم Hash توزیع تقریباً مساوی در طول زمان |
| سیاست به‌ازای هر WAN | هر WAN جدول مسیریابی و قوانین NAT خود را دارد |
| اثبات‌شده در Production | پرکاربردترین روش Multi-WAN روی MikroTik |
| سازگار با Firewall | قوانین Stateful با Connection Mark درست کار می‌کنند |

---

## معایب و ریسک‌ها

| ریسک | جزئیات |
|------|--------|
| Overhead CPU | قوانین Mangle هر Connection جدید را پردازش می‌کنند |
| پیچیدگی پیکربندی | نیاز به هماهنگی Mangle + جداول مسیریابی + NAT |
| توزیع نابرابر لحظه‌ای | Connectionهای کوتاه‌عمر ممکن است روی یک WAN خوشه‌ای شوند |
| بدون وزن‌دهی پهنای باند | 3/0، 3/1، 3/2 تقسیم مساوی می‌دهد (بدون آگاهی از ظرفیت) |
| ترتیب قوانین حیاتی | ترتیب نادرست Mangle طبقه‌بندی را می‌شکند |
| اندازه جدول Connection | تعداد Connection بالا مصرف حافظه را افزایش می‌دهد |

---

## خطاهای رایج پیاده‌سازی

| خطا | پیامد | راه‌حل |
|-----|-------|--------|
| نبود `passthrough=yes` | فقط اولین بسته طبقه‌بندی می‌شود، بقیه بدون Mark | همیشه passthrough=yes روی Connection Mark تنظیم کنید |
| Mangle بعد از Routing | تصمیم مسیریابی قبل از طبقه‌بندی گرفته می‌شود | Mangle باید در prerouting، قبل از Routing باشد |
| بدون جداول مسیریابی مجزا | تمام ترافیک از جدول main استفاده می‌کند | to-WAN1، to-WAN2، to-WAN3 ایجاد کنید |
| NAT بدون فیلتر out-interface | IP اشتباه WAN برای ترجمه | قوانین NAT را به out-interface مشخص تطبیق دهید |
| عدم تطابق Classifier N/M | توزیع نابرابر یا ناقص | N = تعداد WANها، M = ۰ تا N-1 |
| فراموش کردن Bypass Established | طبقه‌بندی مجدد Sessionهای فعال | شرط `connection-mark=no-mark` برای قوانین Classifier |
| بدون check-gateway روی Routeهای PCC | ترافیک به WAN مرده ارسال می‌شود | check-gateway روی تمام Routeهای جدول PCC فعال کنید |

---

**بعدی ←** [Failover](failover.md)
