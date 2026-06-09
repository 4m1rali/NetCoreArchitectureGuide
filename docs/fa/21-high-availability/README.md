# فصل ۲۱ — High Availability (افزونگی روتر)

> حذف روتر به‌عنوان Single Point of Failure در طراحی‌های Multi-WAN.

---

## ۲۱.۱ گزینه‌های معماری HA

| روش | زمان Failover | پیچیدگی | هزینه |
|-----|--------------|---------|-------|
| VRRP (Virtual Router Redundancy) | 1–3 ثانیه | متوسط | 2 روتر |
| Failover دستی Dual router | دقیقه‌ها | پایین | 2 روتر |
| BGP با دو Edge router | ثانیه‌ها | بالا | 2 روتر + BGP |
| Failover Cloud CHR | ثانیه‌ها | متوسط | 2 VM |

---

## ۲۱.۲ VRRP روی MikroTik

```
/interface vrrp
add interface=bridge-lan vrid=1 priority=150 preempt=yes \
    authentication=ah2 password=vrrp-secret

/ip address
add address=192.168.1.1/24 interface=bridge-lan comment="LAN gateway (VRRP virtual)"
```

### طراحی VRRP Dual Router

```
Router-A (Master)                    Router-B (Backup)
├── Priority: 150                    ├── Priority: 100
├── همه لینک‌های WAN فعال            ├── همه لینک‌های WAN فعال
├── PCC + Failover پیکربندی شده      ├── Config یکسان PCC + Failover
├── VRRP Master                      ├── VRRP Backup
└── ترافیک را مدیریت می‌کند          └── در صورت شکست Master تصاحب می‌کند
```

هر دو روتر Config یکسان PCC اجرا می‌کنند. VRRP فقط **IP Gateway LAN** را محافظت می‌کند.

---

## ۲۱.۳ همگام‌سازی Config

از **export/import پیکربندی** یا **The Dude** برای همگام نگه داشتن Configها استفاده کنید:

```
# روی master — export زمان‌بندی‌شده به storage مشترک
/system scheduler
add name=config-sync interval=1h on-event={
    /export file=master-config
    /tool fetch address=192.168.1.200 src-path=master-config.rsc \
        dst-path=backup-config.rsc mode=ftp upload=yes
}
```

---

## ۲۱.۴ ملاحظات WAN HA

| جنبه | جزئیات |
|------|--------|
| هر دو روتر به همه لینک‌های WAN نیاز دارند | کابل فیزیکی به هر روتر یا Switch بین ISP و روترها |
| State NAT همگام نیست | Connectionهای فعال در Failover VRRP drop می‌شوند (~1–3s) |
| State PCC همگام نیست | Connectionهای جدید پس از Failover دوباره توزیع می‌شوند |
| BGP | هر دو روتر می‌توانند Peer باشند — برای WAN HA از BGP به‌جای VRRP استفاده کنید |
| Connection tracking | بین روترها مشترک نیست — از دست رفتن Session انتظار می‌رود |

---

## ۲۱.۵ HA در سطح ISP با BGP

```
Router-A (AS 65050)          Router-B (AS 65050)
├── BGP به ISP-1             ├── BGP به ISP-1
├── BGP به ISP-2             ├── BGP به ISP-2
├── Advertise فضای PI        ├── Advertise فضای PI
└── Active                   └── Standby (prepend بالاتر)

ISP دو مسیر به Prefix شما می‌بیند — Failover خودکار بدون VRRP.
```

---

**فصل بعد →** [فصل ۲۲: Hotspot و Captive Portal](../22-hotspot-captive-portal/README.md)
