# فصل ۱۹ — راهنمای انتخاب سخت‌افزار

> انتخاب سخت‌افزار مناسب MikroTik برای استقرار Production Multi-WAN.

---

## ۱۹.۱ ماتریس انتخاب سخت‌افزار

| مدل | CPU | RAM | پورت‌ها | Throughput | حداکثر کاربر | نقش Multi-WAN |
|-----|-----|-----|--------|------------|--------------|---------------|
| hEX S (RB760iGS) | MMIPS 880MHz | 256MB | 5xGE + SFP | ~400Mbps | 25 | Failover شعبه |
| RB750Gr3 | MMIPS 880MHz | 256MB | 5xGE | ~400Mbps | 25 | Dual-WAN SOHO |
| RB4011iGS+RM | IPQ-4019 4-core | 1GB | 10xGE + SFP+ | ~3.5Gbps | 200 | **استاندارد Enterprise** |
| CCR2004-16G-2S+ | AL32400 4-core | 4GB | 16xGE + 2xSFP+ | ~10Gbps | 500 | Edge ISP |
| CCR2116-12G-4S+ | AL73400 16-core | 16GB | 12xGE + 4xSFP+ | ~25Gbps | 2000 | Core ISP |
| CCR2216-1G-12XS-2XQ | AL73400 16-core | 16GB | 1xGE + 12xSFP+ + 2xQSFP | ~100Gbps | 10000+ | Datacenter |

---

## ۱۹.۲ انتخاب بر اساس سناریو

### SOHO / Home Office (2 WAN، کمتر از ۱۰ کاربر)

| مورد | توصیه |
|------|-------|
| مدل | hEX S یا RB750Gr3 |
| روش | فقط Failover |
| لایسنس | Level 4 |
| هزینه تقریبی | $60–80 |

### SMB / Branch (2–3 WAN، ۵۰ کاربر)

| مورد | توصیه |
|------|-------|
| مدل | RB4011iGS+RM |
| روش | PCC 2-way + Failover |
| لایسنس | Level 4–5 |
| هزینه تقریبی | $250–300 |

### Enterprise HQ (3 WAN، ۲۰۰ کاربر)

| مورد | توصیه |
|------|-------|
| مدل | RB4011 (دوگانه) یا CCR2004 |
| روش | PCC + Policy + QoS |
| لایسنس | Level 5 |
| هزینه تقریبی | $300–800 |

### ISP WISP (3 WAN، 300+ مشترک)

| مورد | توصیه |
|------|-------|
| مدل | CCR2004 یا CCR2116 |
| روش | PCC + PCQ + BGP |
| لایسنس | Level 5–6 |
| هزینه تقریبی | $800–2500 |

### Datacenter Edge (BGP، 10G+)

| مورد | توصیه |
|------|-------|
| مدل | CCR2116 یا CCR2216 |
| روش | BGP + ECMP |
| لایسنس | Level 6 |
| هزینه تقریبی | $2500–5000 |

---

## ۱۹.۳ CPU و کارایی PCC

قوانین Mangle PCC CPU را به‌ازای هر **Connection جدید** مصرف می‌کنند. انتخاب سخت‌افزار باید نرخ Connection را در نظر بگیرد، نه فقط Bandwidth.

| معیار | hEX | RB4011 | CCR2004 | CCR2116 |
|-------|-----|--------|---------|---------|
| Connection جدید/ثانیه | ~500 | ~5,000 | ~20,000 | ~80,000 |
| Connection فعال | 50K | 200K | 500K | 2M+ |
| CPU PCC 3-WAN @ 1Gbps | 60–80% | 15–25% | 5–10% | < 5% |
| WANهای PCC توصیه‌شده | 2 | 3 | 3–4 | 4+ |

**قاعده:** اگر Connectionهای جدید پایدار از 2000/ثانیه بیشتر باشد، از سری CCR استفاده کنید.

---

## ۱۹.۴ الزامات RAM

| جزء | مصرف RAM |
|-----|----------|
| پایه RouterOS | ~50MB |
| Connection tracking (100K entry) | ~200MB |
| BGP full table | ~2–4GB |
| BGP فقط default | ~50MB |
| PCC mangle (بدون RAM اضافه) | فقط CPU |
| User Manager (500 کاربر) | ~100MB |

| استقرار | حداقل RAM |
|---------|-----------|
| Multi-WAN پایه | 256MB |
| Enterprise PCC | 1GB |
| ISP با 500K connection | 4GB |
| BGP full table | 8–16GB |

---

## ۱۹.۵ Storage و Logging

| مدل | Storage | ظرفیت Logging |
|-----|---------|-----------------|
| hEX S | 16MB NAND | حداقل — از remote syslog استفاده کنید |
| RB4011 | 512MB NAND | متوسط |
| CCR2004 | 128MB NAND + USB | خوب با USB disk |
| CCR2116 | 128MB NAND + NVMe slot | عالی |

برای Multi-WAN Production، صرف‌نظر از مدل همیشه از **Remote Syslog** استفاده کنید.

---

## ۱۹.۶ برنامه‌ریزی تعداد Interface

| نوع WAN | Interface مورد نیاز |
|---------|---------------------|
| 2 ISP fiber static | 2 WAN + 1 LAN = 3 |
| 3 ISP mixed | 3 WAN + 1 LAN + 1 mgmt = 5 |
| ISP با VLAN handoff | 1 فیزیکی + VLANها |
| PPPoE × 3 | 3 پورت WAN فیزیکی |
| Dual router VRRP | 2× همه چیز |

**حداقل:** 1 LAN + N WAN + 1 پورت مدیریت

---

## ۱۹.۷ برق افزون و محیط

| الزام | Enterprise | ISP |
|-------|-----------|-----|
| Dual PSU | توصیه می‌شود | الزامی |
| Rack mount | RB4011 1U یا CCR | CCR 1U |
| دما | < 40°C محیط | < 35°C با جریان هوا |
| UPS | الزامی | الزامی با generator |
| Out-of-band mgmt | پورت mgmt اختصاصی | Serial + mgmt اختصاصی |

---

## ۱۹.۸ چک‌لیست سخت‌افزار

| # | مورد | ☐ |
|---|------|---|
| 1 | پورت WAN کافی برای همه ISPها | ☐ |
| 2 | SFP/SFP+ در صورت نیاز fiber WAN | ☐ |
| 3 | RAM کافی برای تعداد Connection | ☐ |
| 4 | CPU کافی برای PCC در Bandwidth هدف | ☐ |
| 5 | سطح لایسنس مطابق الزامات Feature | ☐ |
| 6 | Storage/USB برای logging (یا remote syslog برنامه‌ریزی‌شده) | ☐ |
| 7 | کیت Rack mount برای استقرار Datacenter | ☐ |
| 8 | ظرفیت UPS محاسبه‌شده برای مصرف برق روتر | ☐ |

---

**فصل بعد →** [فصل ۲۰: Migration و Upgrade](../20-migration-upgrade/README.md)
