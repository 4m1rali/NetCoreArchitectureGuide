# پیوست ج — سؤالات متداول (FAQ)

---

## عمومی

### س: آیا می‌توانم Bandwidth سه لینک 100Mbps را به 300Mbps برای یک Download ترکیب کنم؟

**ج:** خیر. Load Balancing **بین Connectionها** توزیع می‌کند، نه درون یک Connection واحد. یک Download از یک WAN استفاده می‌کند (50–100Mbps). اما 10 Download همزمان می‌توانند از هر 3 WAN استفاده کنند (~300Mbps مجموع).

### س: کدام روش را استفاده کنم؟

**ج:** برای محیط‌های NAT (اکثر استقرارها): **PCC + Failover**. برای IP عمومی Routed بدون NAT: **ECMP + Failover**. برای ISP با ASN اختصاصی: **BGP**.

### س: آیا MikroTik Link Bonding واقعی مثل 802.3ad را پشتیبانی می‌کند؟

**ج:** Bonding L2 بومی بین ISPها وجود ندارد. PCC **Load Balancing منطقی** در L3/L4 ارائه می‌دهد. برای Bonding L2 واقعی، هر دو طرف باید آن را پشتیبانی کنند (با ISPها نادر است).

---

## PCC

### س: چرا توزیع PCC من 70/30 است به‌جای 50/50؟

**ج:** PCC **Connectionهای جدید** را Hash می‌کند. اگر بیشتر ترافیک Long-lived باشد (Streaming)، توزیع به زمان شروع Connectionها بستگی دارد. تست‌های کوتاه Skew نشان می‌دهند. در طول ساعت‌ها مانیتور کنید، نه دقیقه‌ها.

### س: آیا می‌توانم PCC را برای سرعت‌های نابرابر WAN Weight کنم؟

**ج:** بله. از Bucketهای Classifier استفاده کنید: `:10/0` تا `:10/6` برای 70% روی WAN1، `:10/7` تا `:10/9` برای 30% روی WAN2.

### س: آیا PCC با IPv6 کار می‌کند؟

**ج:** بله. از `/ipv6 firewall mangle` با همان Syntax Classifier استفاده کنید.

---

## Failover

### س: Failover چقدر سریع است؟

**ج:** معمولاً 3–15 ثانیه با `check-gateway=ping`. Scriptهای Netwatch 10–30 ثانیه اضافه می‌کنند. Failover BGP: 30–90 ثانیه (Hold timer).

### س: آیا Connectionهای فعال از Failover جان سالم به در می‌برند؟

**ج:** خیر. Connectionهای روی WAN شکسته Drop می‌شوند. Connectionهای جدید از WAN پشتیبان استفاده می‌کنند. این رفتار مورد انتظار است.

### س: آیا LTE باید در PCC باشد؟

**ج:** به‌طور کلی **خیر**. LTE را فقط به‌عنوان Failover (distance=3) استفاده کنید تا از هزینه‌های داده غیرمنتظره جلوگیری شود.

---

## NAT

### س: چرا NAT با ECMP می‌شکند؟

**ج:** Hashing per-packet ECMP بسته‌های یک Connection را از WANهای مختلف عبور می‌دهد. ترافیک Return روی WAN اشتباه می‌رسد و Connection Tracking را می‌شکند.

### س: Hairpin NAT چیست؟

**ج:** به Clientهای LAN اجازه می‌دهد IP عمومی شما را از داخل شبکه دسترسی دهند. [فصل ۹](../09-advanced-nat-dns/README.md) را ببینید.

---

## کارایی

### س: CPU روتر من با PCC روی 80% است. چه کنم؟

**ج:** به سخت‌افزار CCR ارتقا دهید، قوانین Mangle را کاهش دهید، Logging غیرضروری را غیرفعال کنید، مطمئن شوید FastTrack خاموش است (با PCC تداخل دارد).

### س: RB4011 با PCC چند Connection را handle می‌کند؟

**ج:** ~200,000 Connection فعال، ~5,000 Connection جدید/ثانیه با Mangle PCC.

---

## لایسنس

### س: برای Multi-WAN چه لایسنسی لازم است؟

**ج:** Level 4 حداقل برای Production. Level 3 فقط برای Failover پایه کار می‌کند. [فصل ۱۸](../18-routeros-licensing/README.md) را ببینید.

### س: آیا CHR رایگان است؟

**ج:** CHR لایسنس رایگان محدود به 1 Mbps دارد. لایسنس‌های پولی ($45–$250) سرعت کامل را باز می‌کنند.

---

## عیب‌یابی

### س: اینترنت فقط روی یک WAN کار می‌کند. PCC متعادل نمی‌کند؟

**ج:** بررسی کنید: آمار mangle (آیا قوانین Hit شده‌اند؟)، وضعیت inactive Route، `passthrough=yes` روی Markها، FastTrack غیرفعال.

### س: پس از Upgrade RouterOS، PCC کار نکرد؟

**ج:** RouterOS 7 از `/routing table` به‌جای `/routing mark` استفاده می‌کند. [فصل ۲۰](../20-migration-upgrade/README.md) را ببینید.

---

**[← Cheat Sheet](cheat-sheet.md)** | **[واژه‌نامه →](glossary.md)**
