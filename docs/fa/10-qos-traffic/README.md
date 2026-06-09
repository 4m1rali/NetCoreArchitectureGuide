# فصل ۱۰ — QoS و اولویت‌بندی ترافیک

> مدیریت پهنای باند و اولویت‌بندی ترافیک در چند لینک WAN.

---

## ۱۰.۱ چرا QoS در Multi-WAN مهم است

بدون QoS، یک دانلود سنگین روی یک WAN می‌تواند لینک را اشباع کند و VoIP، Gaming و ترافیک حیاتی کسب‌وکار را افت دهد — حتی با توزیع PCC کامل.

| مشکل | راه‌حل QoS |
|------|-----------|
| دانلود Bulk از VoIP می‌خورد | صف اولویت‌دار برای UDP 5060/RTP |
| یک کاربر تمام پهنای باند را مصرف می‌کند | محدودیت صف به‌ازای هر کاربر |
| Uplink WAN اشباع شده | PCQ rate limiting به‌ازای زیرشبکه |
| Spike Latency در اوج مصرف | کنترل Burst + اولویت |

---

## ۱۰.۲ انواع صف در MikroTik

| نوع صف | کاربرد | Multi-WAN |
|--------|--------|-----------|
| **sfq** (Stochastic Fairness) | اشتراک عادلانه پیش‌فرض | به‌ازای هر Interface |
| **pcq** (Per Connection) | محدودیت به‌ازای هر کاربر/IP | بهترین برای ISP |
| **red** | اجتناب از ازدحام | اینترفیس‌های WAN |
| **fifo** | سقف ساده پهنای باند | محدودیت کل به‌ازای هر WAN |
| **priority** | VoIP > Web > Bulk | خروجی LAN |

---

## ۱۰.۳ محدودیت پهنای باند به‌ازای هر WAN

```
/queue simple
add name=WAN1-limit target=ether1 max-limit=950M/950M comment="ISP-1 cap"
add name=WAN2-limit target=ether2 max-limit=480M/480M comment="ISP-2 cap"
add name=WAN3-limit target=ether3 max-limit=95M/95M comment="ISP-3 LTE cap"
```

۵٪ فضای خالی زیر نرخ اعلام‌شده ISP بگذارید تا Drop سمت ISP رخ ندهد.

---

## ۱۰.۴ طبقه‌بندی ترافیک مبتنی بر Mangle

```
/ip firewall mangle
add chain=prerouting protocol=udp dst-port=5060,10000-20000 \
    action=mark-packet new-packet-mark=voip-pkt passthrough=yes
add chain=prerouting protocol=tcp dst-port=80,443 \
    action=mark-packet new-packet-mark=web-pkt passthrough=yes
add chain=prerouting protocol=tcp dst-port=21,22 \
    action=mark-packet new-packet-mark=bulk-pkt passthrough=yes
```

---

## ۱۰.۵ درخت صف اولویت‌دار

```
/queue tree
add name=WAN1-root parent=ether1 max-limit=950M
add name=WAN1-voip parent=WAN1-root packet-mark=voip-pkt priority=1 limit-at=50M
add name=WAN1-web parent=WAN1-root packet-mark=web-pkt priority=4 limit-at=200M
add name=WAN1-bulk parent=WAN1-root packet-mark=bulk-pkt priority=8 limit-at=100M
add name=WAN1-default parent=WAN1-root priority=6
```

برای WAN2 و WAN3 با محدودیت‌های متناسب با ظرفیت تکرار کنید.

---

## ۱۰.۶ PCQ عادلانه به‌ازای هر کاربر (ISP)

```
/queue type
add name=pcq-upload kind=pcq pcq-classifier=src-address pcq-rate=10M
add name=pcq-download kind=pcq pcq-classifier=dst-address pcq-rate=50M

/queue simple
add name=user-fair-upload target=ether1 queue=pcq-upload/pcq-upload max-limit=950M
add name=user-fair-download target=bridge-lan queue=pcq-download/pcq-download max-limit=950M
```

---

## ۱۰.۷ یکپارچه‌سازی QoS + PCC

| قانون | دلیل |
|-------|------|
| قوانین Mangle PCC قبل از Markهای QoS | ابتدا طبقه‌بندی Connection |
| QoS روی out-interface (WAN) | Shaping خروجی به‌ازای هر ISP |
| PCC Mark را Shape نکنید | به‌جای آن با packet-mark Shape کنید |
| VoIP روی WAN Policy-routed نگه دارید | QoS + Policy Routing با هم |

---

## ۱۰.۸ Burst و Burst Threshold

```
/queue simple
add name=burst-example target=ether1 max-limit=100M/100M \
    burst-limit=150M/150M burst-threshold=80M/80M burst-time=8s/8s
```

اجازه Burst کوتاه بالاتر از نرخ پایدار — مفید برای الگوی مرور وب.

---

**فصل بعد ←** [فصل ۱۱: VPN روی Multi-WAN](../11-vpn-multiwan/README.md)
