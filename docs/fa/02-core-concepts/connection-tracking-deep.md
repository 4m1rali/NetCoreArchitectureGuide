# Connection Tracking — بررسی عمیق

> رفتار پیشرفته conntrack برای مهندسان Multi-WAN.

---

## ساختار جدول Connection Tracking

| فیلد | ارتباط با Multi-WAN |
|------|---------------------|
| `protocol` | TCP/UDP/ICMP — Timeout به‌ازای هر پروتکل متفاوت است |
| `src-address` / `dst-address` | Endpointهای اصلی قبل از NAT |
| `reply-src-address` / `reply-dst-address` | پس از ترجمه NAT |
| `connection-mark` | تخصیص WAN در PCC — در کل Session پایدار می‌ماند |
| `routing-mark` | جدول مسیریابی فعال برای این Connection |
| `connection-state` | new / established / related / invalid |
| `timeout` | عمر باقی‌مانده Session |
| `orig-packets` / `repl-packets` | حجم ترافیک به‌ازای هر جهت |
| `orig-rate` / `repl-rate` | Throughput لحظه‌ای |
| `helper` | FTP/SIP/H323 ALG — می‌تواند Multi-WAN را مختل کند |
| `fasttrack` | اگر true باشد، mangle Bypass شده است |

---

## تنظیم Timeout برای Multi-WAN

Timeoutهای پیش‌فرض ممکن است برای مقیاس Production ISP/Enterprise بیش‌ازحد تهاجمی باشند.

```
/ip firewall connection tracking
set enabled=yes \
    tcp-established-timeout=1d \
    tcp-time-wait-timeout=10s \
    tcp-close-timeout=10s \
    tcp-syn-sent-timeout=5s \
    tcp-syn-received-timeout=5s \
    udp-timeout=30s \
    udp-stream-timeout=3m \
    icmp-timeout=10s \
    generic-timeout=10m \
    max-entries=1048576
```

| Timeout | اثر خیلی کم | اثر خیلی زیاد |
|---------|-------------|---------------|
| tcp-established | Sessionهای فعال قطع می‌شوند | انباشت حافظه |
| udp | وقفه DNS/VoIP | ورودی‌های کهنه انباشته می‌شوند |
| generic | ابزارهای مبتنی بر ICMP شکست می‌خورند | پر شدن جدول |

---

## برنامه‌ریزی ظرفیت جدول Connection

| کاربران | Connectionهای مورد انتظار | RAM مورد نیاز | max-entries |
|---------|--------------------------|---------------|-------------|
| 50 | 5,000–10,000 | 256MB | 131072 |
| 200 | 20,000–50,000 | 1GB | 262144 |
| 500 | 50,000–150,000 | 4GB | 524288 |
| 1000+ | 150,000–500,000 | 8–16GB | 1048576 |

### مانیتور استفاده از جدول

```
/ip firewall connection print count-only
/ip firewall connection tracking print
/system resource print
```

---

## ALG Helperها و Multi-WAN

Application Layer Gateway (helper) Connectionهای مرتبطی باز می‌کند که باید Connection Mark والد را دنبال کنند.

| Helper | پروتکل | ریسک Multi-WAN |
|--------|--------|----------------|
| FTP | TCP 21 | کانال Data ممکن است WAN اشتباه استفاده کند |
| SIP | UDP 5060 | جریان RTP ممکن است قطع شود |
| H323 | TCP 1720 | پورت‌های پویا با NAT تداخل دارند |
| PPTP | TCP 1723 | پروتکل GRE از Connection Tracking عبور می‌کند |

### غیرفعال‌سازی Helperها وقتی لازم نیست

```
/ip firewall service-port
set ftp disabled=yes
set sip disabled=yes
set h323 disabled=yes
set pptp disabled=yes
```

برای VoIP روی Multi-WAN از **SIP over TCP** یا **محدوده پورت RTP ثابت** با Policy Routing استفاده کنید.

---

## ترافیک Untracked

ترافیک با علامت `untracked` به‌طور کامل از Connection Tracking عبور می‌کند.

```
/ip firewall mangle
add chain=prerouting action=mark-connection new-connection-mark=\
    no-mark passthrough=yes connection-state=established
```

**هرگز برای ترافیک PCC-balanced از untracked استفاده نکنید** — Markها پایدار نمی‌مانند.

---

## تأیید پایداری Connection Mark

```
/ip firewall connection print where src-address~"192.168.1" \
    and connection-mark!=""
```

هر Connection فعال LAN در استقرارهای PCC باید `connection-mark` غیرخالی نشان دهد.

---

**فصل بعد ←** [فصل ۳: جدول مقایسه](../03-comparison-table/README.md)
