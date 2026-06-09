# تمرین‌های آزمایشگاهی — تمرین Multi-WAN

> ۱۲ تمرین آزمایشگاهی ایزوله. از CHR یا سخت‌افزار یدکی استفاده کنید. هرگز روی Production.

---

## راه‌اندازی محیط آزمایشگاهی

```
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│  Virtual ISP1│   │  Virtual ISP2│   │  Virtual ISP3│
│  203.0.113.0 │   │ 198.51.100.0 │   │  192.0.2.0   │
└──────┬───────┘   └──────┬───────┘   └──────┬───────┘
       │ ether1           │ ether2           │ ether3
       └──────────┬───────┴──────────────────┘
                  │
           ┌──────▼──────┐
           │  CHR Router │
           │  RouterOS 7 │
           └──────┬──────┘
                  │ ether4
           ┌──────▼──────┐
           │  Test LAN   │
           │ 192.168.1.0 │
           └─────────────┘
```

### پیکربندی پایه (تمام آزمایشگاه‌ها)

```
/interface list
add name=WAN
add name=LAN
/interface list member
add interface=ether1 list=WAN
add interface=ether2 list=WAN
add interface=ether3 list=WAN
add interface=ether4 list=LAN

/ip address
add address=203.0.113.2/30 interface=ether1
add address=198.51.100.2/30 interface=ether2
add address=192.0.2.2/30 interface=ether3
add address=192.168.1.1/24 interface=ether4
```

---

## آزمایشگاه ۱ — PCC پایه ۳-WAN

**هدف:** پیکربندی PCC و تأیید توزیع تقریباً ۳۳/۳۳/۳۳.

**وظایف:**
1. ایجاد جداول مسیریابی `to-WAN1`، `to-WAN2`، `to-WAN3`
2. افزودن Routeهای PCC با check-gateway
3. پیکربندی قوانین Mangle PCC (3/0، 3/1، 3/2)
4. افزودن Masquerade NAT به‌ازای هر WAN
5. تولید ترافیک از ۳ کلاینت LAN به‌طور همزمان

**تأیید:**
```
/ip firewall mangle print stats
/ip firewall connection print count-only where connection-mark="WAN1-conn"
/ip firewall connection print count-only where connection-mark="WAN2-conn"
/ip firewall connection print count-only where connection-mark="WAN3-conn"
/interface monitor-traffic ether1,ether2,ether3 once
```

**معیار موفقیت:** هر WAN بین ۲۵ تا ۴۰٪ Connectionهای جدید را حمل کند.

---

## آزمایشگاه ۲ — شبیه‌سازی Failover

**هدف:** تأیید Failover خودکار هنگام از کار افتادن یک WAN.

**وظایف:**
1. تکمیل پیکربندی آزمایشگاه ۱
2. تولید Ping مداوم از LAN: `/ping 8.8.8.8 count=1000`
3. غیرفعال‌سازی ether1: `/interface disable ether1`
4. اندازه‌گیری Downtime
5. فعال‌سازی مجدد ether1 و مشاهده Recovery

**تأیید:**
```
/log print where message~"route"
/tool netwatch print
```

**معیار موفقیت:** Failover کمتر از ۱۵ ثانیه. از دست رفتن Ping کمتر از ۲۰ بسته.

---

## آزمایشگاه ۳ — تست تقارن NAT

**هدف:** اثبات اینکه NAT به‌ازای هر WAN مسیر بازگشت را حفظ می‌کند.

**وظایف:**
1. پیکربندی PCC (آزمایشگاه ۱)
2. از کلاینت LAN، ۱۰ Session HTTPS به سایت‌های مختلف باز کنید
3. جدول Connection را برای سازگاری ترجمه NAT بررسی کنید

**تأیید:**
```
/ip firewall connection print where protocol=tcp and dst-port=443
```

**معیار موفقیت:** هر Connection دارای `repl-src-address` ثابتی باشد که با IP عمومی صحیح WAN مطابقت دارد.

---

## آزمایشگاه ۴ — PCC وزن‌دار (۷۰/۳۰)

**هدف:** پیکربندی توزیع بار نابرابر.

**وظایف:**
1. از `:10/0` تا `:10/6` برای WAN1 و `:10/7` تا `:10/9` برای WAN2 استفاده کنید
2. WAN3 را برای این آزمایشگاه غیرفعال کنید
3. ۱۰۰ Connection TCP جدید تولید کنید

**معیار موفقیت:** WAN1 ≈ ۶۵–۷۵٪ Connectionها، WAN2 ≈ ۲۵–۳۵٪.

---

## آزمایشگاه ۵ — Policy Routing (اولویت VoIP)

**هدف:** اجبار Subnet VoIP به WAN1، بقیه از طریق PCC.

**وظایف:**
1. محدوده آدرس 192.168.1.240/28 را به‌عنوان «Subnet VoIP» ایجاد کنید
2. قانون Mangle: UDP 5060، 10000-20000 → WAN1-conn (بالای قوانین PCC)
3. ترافیک SIP از .240/28 و HTTP از .10-.50 تولید کنید
4. تأیید کنید VoIP همیشه از WAN1 استفاده می‌کند

**معیار موفقیت:** ۱۰۰٪ Connectionهای UDP 5060 با Mark WAN1-conn مشخص شده‌اند.

---

## آزمایشگاه ۶ — ممیزی سخت‌سازی امنیتی

**هدف:** یافتن و بستن سرویس‌های در معرض دید.

**وظایف:**
1. اجرای `/ip service print`
2. اجرای `/user print`
3. اسکن روتر از سمت «WAN» (Nmap از شبکه ISP مجازی):
   - `nmap -sS -p 21,22,23,80,443,8291,8728,8729 <router-wan-ip>`
4. اعمال سخت‌سازی از [فصل ۱۶](../16-security-hardening/README.md)
5. اسکن مجدد — تمام Portها از WAN فیلتر/بسته باشند

**معیار موفقیت:** صفر Port مدیریت باز از WAN.

---

## آزمایشگاه ۷ — آگاهی CVE-2018-14847 (دفاعی)

**هدف:** درک آسیب‌پذیری Winbox و تأیید وضعیت Patch.

> **CVE-2018-14847** امکان خواندن فایل‌های دلخواه از طریق Port Winbox را فراهم می‌کرد. در RouterOS 6.42.1+ و 6.43+ رفع شد.

**وظایف:**
1. استقرار نمونه CHR با RouterOS قدیمی 6.40.x (فقط آزمایشگاه ایزوله)
2. بررسی نسخه: `/system resource print`
3. تأیید امکان خواندن فایل Winbox در نسخه بدون Patch (فقط تحقیق)
4. ارتقا به آخرین long-term: `/system package update`
5. تست مجدد — آسیب‌پذیری Patch شده

**معیار موفقیت:** درک اینکه چرا **هرگز** Winbox را به WAN در معرض دید قرار ندهید و **همیشه** Patch کنید.

---

## آزمایشگاه ۸ — تست تزریق Script در Netwatch

**هدف:** کشف Scriptهای ناامن Netwatch.

**وظایف:**
1. بررسی تمام ورودی‌های Netwatch: `/tool netwatch print`
2. بررسی اینکه آیا Scriptها ورودی خارجی می‌پذیرند
3. جایگزینی Scriptهای ناامن با نسخه‌های Parameterized امن
4. تست اینکه Failover پس از سخت‌سازی Script همچنان کار می‌کند

**الگوی ناامن:**
```
down-script="/ip route disable [find gateway=203.0.113.1]"
```

**الگوی امن‌تر:**
```
down-script={/ip route disable [find comment="PCC ISP-1"]}
```

**معیار موفقیت:** هیچ Script با IP سخت‌کدشده که قابل دستکاری باشد وجود نداشته باشد.

---

## آزمایشگاه ۹ — شکاف Firewall IPv6

**هدف:** اثبات اینکه Firewall فقط IPv4، IPv6 را در معرض دید قرار می‌دهد.

**وظایف:**
1. پیکربندی IPv6 روی تمام Interfaceها
2. اعمال Firewall فقط IPv4 (بدون `/ipv6 firewall`)
3. تلاش برای دسترسی از WAN از طریق IPv6
4. افزودن قوانین `/ipv6 firewall filter` مشابه IPv4
5. تست مجدد — دسترسی مسدود شود

**معیار موفقیت:** ورودی IPv6 از WAN پس از افزودن Firewall IPv6 Drop شود.

---

## آزمایشگاه ۱۰ — اشباع جدول Connection

**هدف:** درک محدودیت‌های منابع.

**وظایف:**
1. تنظیم `max-entries=10000` (فقط آزمایشگاه)
2. تولید Connection با `/tool traffic-generator` یا flood ping
3. مانیتور: `/ip firewall connection print count-only`
4. مشاهده رفتار در محدودیت — Connectionهای جدید Drop می‌شوند؟
5. بازگرداندن `max-entries=262144`

**معیار موفقیت:** ثبت CPU و RAM در ۸۰٪ ظرفیت جدول Connection.

---

## آزمایشگاه ۱۱ — افشای Secret در فایل Backup

**هدف:** یادگیری اینکه Exportهای `.rsc` شامل رمزهای Plaintext هستند.

**وظایف:**
1. `/export file=test-backup`
2. باز کردن فایل — جستجو برای `password=`، `secret=`، `pre-shared-key=`
3. پیاده‌سازی Redaction رمز قبل از ذخیره Backup خارج از روتر
4. مقایسه `/system backup save` (باینری، کمی بهتر) با export

**معیار موفقیت:** سیاست تیم: هرگز Exportهای `.rsc` را بدون رمزگذاری خارج از روتر ذخیره نکنید.

---

## آزمایشگاه ۱۲ — یکپارچه‌سازی کامل Multi-WAN + امنیت

**هدف:** آزمایشگاه آماده Production ترکیب تمام مهارت‌ها.

**وظایف:**
1. PCC 3-WAN + Failover + NAT به‌ازای هر WAN
2. Policy Routing برای Subnet مدیریت
3. سخت‌سازی امنیتی کامل ([فصل ۱۶](../16-security-hardening/README.md) + [فصل ۲۵](README.md))
4. پیکربندی مانیتورینگ SNMP
5. Script Backup روزانه خودکار
6. مستندسازی کل Config با طرح IP
7. شبیه‌سازی خرابی WAN1 در حین ترافیک فعال
8. اجرای اسکن امنیتی از WAN — صفر Port باز
9. Export Config نهایی به‌عنوان قالب تحویلی

**معیار موفقیت:** عبور از تمام دستورات تأیید فصل‌های ۵، ۶ و ۱۶.

---

## Rubric امتیازدهی آزمایشگاه

| امتیاز | معیار |
|--------|-------|
| **Expert** | تمام ۱۲ آزمایشگاه تکمیل، مستند و اسکن امنیتی پاک |
| **Advanced** | آزمایشگاه‌های ۱–۸ تکمیل، Failover کمتر از ۱۰ ثانیه |
| **Intermediate** | آزمایشگاه‌های ۱–۵ تکمیل، PCC متعادل |
| **Beginner** | آزمایشگاه‌های ۱–۲ تکمیل |

---

**بعدی ←** [مرجع CVE و Exploit](cve-exploit-reference.md)
