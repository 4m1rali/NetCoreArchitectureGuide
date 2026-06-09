# پیوست ه — راهنمای Wireless WAN و LTE

> LTE/5G و WISP بی‌سیم به‌عنوان لینک Multi-WAN.

---

## LTE به‌عنوان WAN پشتیبان

```
/interface lte apn
set [find] apn=internet authentication=chap password=secret user=lteuser

/interface lte
set [find] disabled=no

/ip dhcp-client
add interface=lte1 add-default-route=no use-peer-dns=no

/ip route
add dst-address=0.0.0.0/0 gateway=lte1 distance=3 check-gateway=ping comment="LTE backup only"
```

**هرگز LTE را به Classifier PCC اضافه نکنید.**

---

## مانیتورینگ LTE

```
/tool netwatch
add host=8.8.8.8 interval=30s timeout=5s \
    down-script={
        :log warning "Primary WANs down — LTE should be active"
    }

/interface lte monitor lte1 once
```

---

## آستانه‌های کیفیت سیگنال

| RSSI (dBm) | کیفیت | اقدام |
|------------|-------|-------|
| -50 تا -70 | عالی | Backup Production مناسب |
| -70 تا -85 | خوب | برای Backup قابل قبول |
| -85 تا -100 | متوسط | نزدیک مانیتور کنید |
| < -100 | ضعیف | به این لینک تکیه نکنید |

```
/interface lte monitor lte1 once
# بررسی: rssi, rsrp, sinr
```

---

## WISP بی‌سیم به‌عنوان WAN

```
/interface wireless
set wlan1 mode=station ssid=WISP-AP frequency=5180 band=5ghz-a/n/ac

/ip dhcp-client
add interface=wlan1 add-default-route=no use-peer-dns=no
```

از رادیو 5GHz اختصاصی برای WISP backhaul استفاده کنید؛ جدا از WiFi AP محلی.

---

## کنترل مصرف داده روی LTE

```
/queue simple
add name=lte-cap target=lte1 max-limit=50M/50M comment="Cap LTE bandwidth"

/ip firewall address-list
add list=blocked-on-lte address=0.0.0.0/0

/ip firewall mangle
add chain=prerouting out-interface=lte1 connection-bytes=500000000-0 \
    action=mark-connection new-connection-mark=block-lte passthrough=yes

/ip firewall filter
add chain=forward connection-mark=block-lte action=drop comment="Block large transfers on LTE"
```

---

**[← اشتباهات رایج](common-mistakes.md)**
