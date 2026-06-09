# Failover

> بازیابی خودکار مسیر WAN و افزونگی Gateway.

---

## تعریف مهندسی

**Failover** مکانیزم تاب‌آوری شبکه است که سلامت Gatewayهای WAN را مانیتور می‌کند و هنگام غیرقابل‌دسترس شدن Gateway اصلی، ترافیک را به‌طور خودکار به مسیر پشتیبان هدایت می‌کند. در MikroTik، Failover از طریق **مانیتورینگ Gateway (check-gateway)** همراه با **Recursive Routing** و **اولویت‌بندی Route Distance** پیاده‌سازی می‌شود.

Failover در **لایه ۳** عمل می‌کند و مستقل از Load Balancing است — تداوم را تضمین می‌کند، نه توزیع.

---

## جریان داخلی روتر (گام‌به‌گام)

```
1. Static route configured with check-gateway=ping
   → Route: 0.0.0.0/0 via 203.0.113.1 distance=1 check-gateway=ping
2. Router sends ICMP ping to 203.0.113.1 every check-interval
3. Gateway responds → route status: ACTIVE
4. Traffic flows through ISP-1 normally
5. ISP-1 link fails (cable cut, ISP outage, gateway down)
6. Ping timeout → route status: INACTIVE
7. Router selects next best route:
   → 0.0.0.0/0 via 198.51.100.1 distance=2 check-gateway=ping
8. NEW connections use ISP-2
9. EXISTING connections on ISP-1: DROPPED (expected behavior)
10. ISP-1 recovers → ping success → route ACTIVE again
11. New connections can use ISP-1 (if distance=1 is preferred)
```

---

## رفتار در MikroTik RouterOS

### پیکربندی پایه Failover

```
/ip route
add dst-address=0.0.0.0/0 gateway=203.0.113.1 distance=1 check-gateway=ping
add dst-address=0.0.0.0/0 gateway=198.51.100.1 distance=2 check-gateway=ping
add dst-address=0.0.0.0/0 gateway=192.0.2.1 distance=3 check-gateway=ping
```

### گزینه‌های check-gateway

| گزینه | رفتار |
|-------|-------|
| `ping` | ICMP ping به IP Gateway — رایج‌ترین |
| `arp` | بررسی ARP Resolution — برای Gatewayهای مستقیم |
| `none` | بدون مانیتورینگ — Route همیشه فعال (توصیه نمی‌شود) |

### Recursive Routing برای Failover

وقتی Gateway مستقیماً قابل‌دسترس نیست (مثلاً PPPoE یا Metro Ethernet):

```
/ip route
add dst-address=203.0.113.1/32 gateway=ether1 scope=10
add dst-address=0.0.0.0/0 gateway=203.0.113.1 distance=1 check-gateway=ping
```

**جریان:**
1. مسیر به 203.0.113.1 از طریق ether1 حل می‌شود (scope=10)
2. Default Route از 203.0.113.1 به‌عنوان Gateway استفاده می‌کند
3. check-gateway به 203.0.113.1 ping می‌زند
4. اگر غیرقابل‌دسترس باشد، هر دو Route Inactive می‌شوند

### Failover همراه PCC

در Production، Failover **در کنار** PCC کار می‌کند:

- Routeهای PCC در جدول `to-WAN1` دارای check-gateway روی Gateway ISP-1 هستند
- با شکست ISP-1، Route جدول `to-WAN1` Inactive می‌شود
- ترافیک PCC طبقه‌بندی‌شده برای WAN1 مسیر فعال ندارد → **Drop می‌شود**
- **راه‌حل:** Routeهای Fallback اضافه کنید یا از اسکریپت Netwatch برای حذف Markهای PCC برای WAN شکست‌خورده استفاده کنید

### Failover پیشرفته با Netwatch

```
/tool netwatch
add host=203.0.113.1 interval=10s timeout=3s \
    up-script="/ip route enable [find gateway=203.0.113.1]" \
    down-script="/ip route disable [find gateway=203.0.113.1]"
```

---

## موارد استفاده

| محیط | کاربرد |
|------|--------|
| دفتر مرکزی Enterprise | فیبر اصلی + Failover LTE پشتیبان |
| لبه ISP | افزونگی ارائه‌دهنده Upstream |
| دفتر شعبه | Dual ISP با سوئیچ خودکار |
| سرویس‌های حیاتی | Gateway VoIP نباید اتصال را از دست بدهد |
| هر Multi-WAN | همراه اجباری PCC یا ECMP |

---

## مزایا

| مزیت | جزئیات |
|------|--------|
| بازیابی خودکار | بدون مداخله دستی |
| پیکربندی ساده | Distance + check-gateway کافی است |
| تشخیص سریع | Timeout Ping معمولاً ۳–۱۰ ثانیه |
| سازگار با هر روش | با PCC، ECMP یا مستقل |
| قابلیت اطمینان اثبات‌شده | رویکرد استاندارد در تمام فروشندگان روتر |

---

## معایب و ریسک‌ها

| ریسک | جزئیات |
|------|--------|
| از دست رفتن Session فعال | Connectionهای موجود روی WAN شکست‌خورده Drop می‌شوند |
| Flapping | لینک ناپایدار باعث Failover/Failback مکرر می‌شود |
| مشکل DNS Cache | کلاینت‌ها ممکن است پاسخ DNS WAN شکست‌خورده را Cache کنند |
| Connectionهای یتیم PCC | Connectionهای Markشده برای WAN مرده مسیر ندارند |
| مسیریابی نامتقارن در بازیابی | ترافیک بازگشتی ممکن است قبل از بازیابی Route برسد |
| نقطه واحد مانیتورینگ | Ping به Gateway شکست سمت ISP را تشخیص نمی‌دهد |

### تشخیص شکست سمت ISP

Ping به Gateway فقط دسترسی L2/L3 به Gateway ISP را تأیید می‌کند — نه اتصال اینترنت فراتر از آن.

**راه‌حل:** یک Host خارجی را مانیتور کنید:

```
/tool netwatch
add host=8.8.8.8 interval=10s timeout=3s \
    up-script="..." down-script="..."
```

یا از Recursive Routing با چند Target مانیتور استفاده کنید.

---

## خطاهای رایج پیاده‌سازی

| خطا | پیامد | راه‌حل |
|-----|-------|--------|
| بدون check-gateway | Routeهای مرده برای همیشه فعال می‌مانند | همیشه check-gateway=ping فعال کنید |
| Distance یکسان روی تمام Routeها | ترتیب Failover غیرقابل‌پیش‌بینی | از distance=1,2,3 برای اولویت استفاده کنید |
| نبود Route Recursive | Gateway غیرقابل‌دسترس، Route هرگز فعال نمی‌شود | Host Route به Gateway از طریق Interface اضافه کنید |
| Failover بدون Fallback PCC | ترافیک PCC در شکست WAN Blackhole می‌شود | اسکریپت Netwatch پیاده‌سازی یا Markهای PCC غیرفعال کنید |
| Interval Ping بیش‌ازحد تهاجمی | مثبت کاذب روی لینک‌های شلوغ | حداقل interval=10s timeout=3s |
| بدون مانیتورینگ Upstream | Gateway بالا اما Routing ISP شکسته | IP خارجی (8.8.8.8، 1.1.1.1) را مانیتور کنید |

---

**بعدی ←** [Load Balancing](load-balancing.md)
