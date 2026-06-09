# فصل ۲۴ — Disaster Recovery و Backup

> برنامه‌ریزی Business Continuity برای Edge routerهای Multi-WAN.

---

## ۲۴.۱ استراتژی Backup

| نوع | فرکانس | Storage | زمان Recovery |
|-----|--------|---------|---------------|
| Config export (.rsc) | روزانه | FTP + offsite | 5–15 دقیقه |
| Binary backup (.backup) | هفتگی | USB + offsite | 2–5 دقیقه |
| Full Netinstall image | ماهانه | دستگاه Lab | 30–60 دقیقه |
| مستندات (طرح IP) | در صورت تغییر | Git/wiki | مرجع |

---

## ۲۴.۲ Script Backup خودکار

```
/system script
add name=full-backup source={
    :local date [/system clock get date]
    :local time [/system clock get time]
    :local name ("backup-" . $date . "-" . $time)
    /export file=$name
    /system backup save name=$name
    :log info ("Backup created: $name")
}

/system scheduler
add name=backup-daily interval=1d on-event=full-backup start-time=01:00:00
add name=backup-weekly interval=7d on-event=full-backup start-time=01:30:00
```

---

## ۲۴.۳ رویه‌های Recovery

### سناریو A — خرابی Config

```
/import file=latest-backup.rsc
# تغییرات را بررسی کنید، در صورت نیاز reboot
/system reboot
```

### سناریو B — خرابی سخت‌افزار (جایگزینی همان مدل)

```
1. نصب RouterOS همان نسخه روی سخت‌افزار جدید
2. اعمال License key برای software-id جدید
3. /import file=latest-backup.rsc
4. تأیید لینک‌های WAN، Routeها، PCC
5. به‌روزرسانی ARP روی Switchهای upstream در صورت تغییر MAC
```

### سناریو C — از دست رفتن کامل (سخت‌افزار متفاوت)

```
1. نصب RouterOS روی سخت‌افزار موجود
2. پیکربندی دستی Interfaceها (نام‌ها ممکن است متفاوت باشند)
3. Import backup — رفع ناسازگاری نام Interface
4. تست همه مسیرهای WAN قبل از Cutover Production
```

### سناریو D — Ransomware / Compromise

```
1. Netinstall (پاک‌سازی کامل دستگاه)
2. نصب آخرین RouterOS
3. Import backup معتبر از قبل Compromise
4. تغییر همه Passwordها
5. بررسی قوانین Firewall برای Backdoor
6. غیرفعال‌سازی کلیدهای VPN Compromise شده
```

---

## ۲۴.۴ مستنداتی که باید Off-Router نگه دارید

| سند | محتوا |
|-----|-------|
| طرح IP | همه IP/Gateway/Subnetهای WAN/LAN |
| لیست تماس ISP | شماره حساب، تلفن NOC، Circuit ID |
| Password Vault | Admin، VPN، SNMP، Credentialهای API |
| دیاگرام شبکه | توپولوژی فیزیکی با نگاشت پورت |
| Log تغییر Config | تاریخ، تغییر، مهندس، دلیل |
| کلیدهای License | نگاشت Software-ID → License key |

---

## ۲۴.۵ اهداف Recovery Time

| سناریو | هدف RTO | هدف RPO |
|--------|---------|---------|
| خطای Config | 15 دقیقه | 0 (rollback) |
| خرابی سخت‌افزار | 1 ساعت | 24 ساعت (backup روزانه) |
| قطعی ISP (تکی) | 10 ثانیه (failover) | 0 |
| قطعی ISP (همه) | 4+ ساعت (تعمیر ISP) | N/A |
| Disaster کل سایت | 4–8 ساعت | 24 ساعت |

---

## ۲۴.۶ استراتژی Spare Hardware

| Tier | Spare | Storage |
|------|-------|---------|
| SOHO | همان مدل روی قفسه | دفتر |
| Enterprise | Cold spare از پیش پیکربندی‌شده | همان ساختمان |
| ISP | Hot standby router (VRRP) | همان Rack |
| Datacenter | CCR یکسان Pre-staged | همان DC |

---

**بعد →** [پیوست ج: FAQ](../appendix/faq.md)
