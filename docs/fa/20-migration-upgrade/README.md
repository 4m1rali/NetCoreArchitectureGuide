# فصل ۲۰ — Migration و Upgrade (RouterOS 6 → 7)

> مسیر Migration امن برای استقرارهای Multi-WAN موجود که به RouterOS 7 ارتقا می‌دهند.

---

## ۲۰.۱ چرا به RouterOS 7 Migrate کنیم

| Feature | ROS 6 | ROS 7 |
|---------|-------|-------|
| جداول مسیریابی | `/routing mark` | `/routing table` (تمیزتر) |
| BGP | مدل instance قدیمی | مدل Template + Connection |
| ECMP per-connection | در دسترس نیست | `ecmp-per-connection=yes` |
| WireGuard | محدود | Native، پشتیبانی کامل |
| VRF | پایه | پشتیبانی کامل VRF |
| IPv6 firewall | مشترک با IPv4 | جداگانه `/ipv6 firewall` |
| Container | خیر | Container شبیه Docker |
| کارایی | خوب | 20–40% بهتر روی همان سخت‌افزار |
| پشتیبانی بلندمدت | حالت Maintenance | توسعه فعال |

---

## ۲۰.۲ چک‌لیست Pre-Migration

| # | وظیفه | دستور |
|---|-------|-------|
| 1 | Export کامل Config | `/export file=pre-migration-backup` |
| 2 | Binary backup | `/system backup save name=pre-migration` |
| 3 | مستندسازی Routeهای فعلی | `/ip route print detail` |
| 4 | مستندسازی قوانین mangle | `/ip firewall mangle print` |
| 5 | مستندسازی قوانین NAT | `/ip firewall nat print` |
| 6 | ثبت سطح لایسنس | `/system license print` |
| 7 | تست در Lab اول | هرگز Production را مستقیم Upgrade نکنید |
| 8 | زمان‌بندی پنجره Maintenance | 30–60 دقیقه |

---

## ۲۰.۳ تغییرات کلیدی پیکربندی

### جداول مسیریابی (بزرگ‌ترین تغییر)

```
# RouterOS 6
/ip route rule
add src-address=192.168.1.0/24 action=lookup routing-mark=to-WAN1

/routing mark
add name=to-WAN1

/ip route
add dst-address=0.0.0.0/0 gateway=GW1 routing-mark=to-WAN1

# RouterOS 7
/routing table
add name=to-WAN1 fib

/ip route
add dst-address=0.0.0.0/0 gateway=GW1 routing-table=to-WAN1

# Mangle بدون تغییر می‌ماند
add chain=prerouting connection-mark=WAN1-conn action=mark-routing \
    new-routing-mark=to-WAN1 passthrough=yes
```

### BGP (طراحی مجدد کامل)

```
# RouterOS 6
/routing bgp instance
/routing bgp peer
/routing bgp network

# RouterOS 7
/routing bgp template
/routing bgp connection
/routing bgp network
```

### Firewall

```
# RouterOS 7 — IPv6 firewall جداگانه است
/ipv6 firewall filter
/ipv6 firewall nat
/ipv6 firewall mangle
```

---

## ۲۰.۴ رویه Migration

```
STEP 1: تست Lab با Config export شده
STEP 2: Upgrade RouterOS (هنوز RouterBOARD firmware نه)
        /system package update check-for-updates
        /system package update download
        /system reboot
STEP 3: تأیید auto-migration پیکربندی
        /ip route print detail
        /routing table print
STEP 4: اصلاح دستی قوانین شکسته
STEP 5: تست توزیع PCC
        /ip firewall mangle print stats
STEP 6: تست Failover (قطع یک WAN)
STEP 7: تست NAT
        /ip firewall connection print
STEP 8: Upgrade RouterBOARD firmware در صورت پایداری
        /system routerboard upgrade
        /system reboot
STEP 9: مانیتور 24 ساعت قبل از بستن Maintenance
```

---

## ۲۰.۵ طرح Rollback

```
# اگر Migration شکست خورد:
1. Netinstall با RouterOS 6.x
2. Import backup pre-migration:
   /import file=pre-migration-backup.rsc
3. یا Restore binary backup:
   /system backup load name=pre-migration
```

Backup pre-migration را روی لپ‌تاپ و USB نگه دارید.

---

## ۲۰.۶ مشکلات رایج Migration

| مشکل | علت | رفع |
|------|-----|-----|
| PCC کار نمی‌کند | routing-mark در مقابل routing-table | ایجاد entryهای `/routing table` |
| Sessionهای BGP down | فرمت Config BGP جدید | پیکربندی مجدد با مدل template |
| IPv6 firewall گم شده | جدا از IPv4 | بازسازی در `/ipv6 firewall` |
| FastTrack PCC را می‌شکند | تغییر رفتار | غیرفعال‌سازی FastTrack |
| Config Wireless از دست رفته | پکیج Wireless جداگانه | نصب پکیج wireless در ROS 7 |
| Scriptها fail می‌کنند | تغییر Syntax | بررسی و به‌روزرسانی scriptها |

---

## ۲۰.۷ Migration بدون Downtime (Dual Router)

برای شبکه‌های Production بحرانی:

```
1. استقرار روتر دوم با RouterOS 7 + Config جدید
2. تست کامل روی پورت‌های جداگانه
3. انتقال IP Gateway LAN به روتر جدید
4. روتر قدیمی Hot Standby می‌شود
5. روتر قدیمی را 30 روز برای Rollback نگه دارید
```

---

**فصل بعد →** [فصل ۲۱: High Availability](../21-high-availability/README.md)
