# فصل ۴ — طراحی Production

> معماری واقعی Multi-WAN با ۳ ISP و تحلیل کامل جریان ترافیک.

---

## ۴.۱ تعریف سناریو

### زیرساخت

| جزء | جزئیات |
|-----|--------|
| روتر | MikroTik CCR2004-16G-2S+ (یا RB4011) |
| RouterOS | 7.x |
| ISP-1 | فیبر ۱Gbps — اصلی (Latency پایین) |
| ISP-2 | Metro Ethernet 500Mbps — ثانویه |
| ISP-3 | LTE پشتیبان 100Mbps — سوم |
| LAN | 192.168.1.0/24 — ۲۰۰ کاربر + سرورها |
| سرویس‌ها | وب‌سرور، VoIP، فایل‌سرور، ۲۰۰ ایستگاه کاری |

### طرح آدرس IP

| اینترفیس | آدرس IP | شبکه |
|----------|---------|------|
| ether1 (WAN1/ISP-1) | 203.0.113.2/30 | 203.0.113.0/30 |
| ether2 (WAN2/ISP-2) | 198.51.100.2/30 | 198.51.100.0/30 |
| ether3 (WAN3/ISP-3) | 192.0.2.2/30 | 192.0.2.0/30 |
| ether5 (LAN) | 192.168.1.1/24 | 192.168.1.0/24 |

| Gateway | آدرس IP |
|---------|---------|
| ISP-1 GW | 203.0.113.1 |
| ISP-2 GW | 198.51.100.1 |
| ISP-3 GW | 192.0.2.1 |

---

## ۴.۲ توپولوژی شبکه

```
                          INTERNET
                             │
          ┌──────────────────┼──────────────────┐
          │                  │                  │
    ┌─────┴─────┐     ┌───────┴──────┐    ┌─────┴─────┐
    │   ISP-1   │     │    ISP-2     │    │   ISP-3   │
    │  Fiber 1G │     │  Metro 500M  │    │  LTE 100M │
    │203.0.113.1│     │198.51.100.1  │    │ 192.0.2.1 │
    └─────┬─────┘     └───────┬──────┘    └─────┬─────┘
          │ ether1            │ ether2          │ ether3
          │                   │                 │
    ┌─────┴───────────────────┴─────────────────┴─────┐
    │                                                    │
    │              MikroTik CCR2004                      │
    │              RouterOS 7.x                          │
    │                                                    │
    │   ┌─────────┐  ┌──────────┐  ┌──────────────┐    │
    │   │ Mangle  │  │ Routing  │  │   Firewall   │    │
    │   │  PCC    │  │  Tables  │  │  NAT + Filter│    │
    │   └─────────┘  └──────────┘  └──────────────┘    │
    │                                                    │
    │                    ether5                          │
    └────────────────────┬───────────────────────────────┘
                         │
              ┌──────────┴──────────┐
              │    LAN Switch       │
              │  192.168.1.0/24    │
              └──────────┬──────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
   ┌────┴────┐    ┌──────┴──────┐   ┌────┴────┐
   │  Users  │    │ Web Server  │   │  VoIP   │
   │ .10-.200│    │  .10:80/443 │   │  .250   │
   └─────────┘    └─────────────┘   └─────────┘
```

---

## ۴.۳ دیاگرام جریان ترافیک

### جریان ترافیک خروجی (PCC)

```
[Client 192.168.1.50]
        │
        ▼
[ether5 LAN] ──→ [Bridge/LAN interface-list]
        │
        ▼
[Firewall FILTER: forward] ── accept from LAN
        │
        ▼
[Mangle PREROUTING]
   ├── PCC classifier → connection-mark: WAN2-conn
   └── routing-mark: to-WAN2
        │
        ▼
[Connection Tracking] ── CREATE entry
        │
        ▼
[Routing Table: to-WAN2]
   └── 0.0.0.0/0 via 198.51.100.1 → ACTIVE
        │
        ▼
[NAT SRCNAT] ── masquerade out-interface=ether2
   └── 192.168.1.50 → 198.51.100.2
        │
        ▼
[ether2 WAN2] ──→ [ISP-2] ──→ [Internet]
```

### جریان ترافیک ورودی (بازگشت)

```
[Internet] ──→ [ISP-2] ──→ [ether2]
        │
        ▼
[Connection Tracking] ── LOOKUP → ESTABLISHED
   └── connection-mark: WAN2-conn
        │
        ▼
[NAT DSTNAT reverse] ── 198.51.100.2 → 192.168.1.50
        │
        ▼
[Routing] ── 192.168.1.0/24 via ether5 (connected)
        │
        ▼
[ether5 LAN] ──→ [Client 192.168.1.50]
```

---

## ۴.۴ جریان تصمیم مسیریابی

```
                    ┌─────────────────┐
                    │  Packet Arrives  │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ Connection-mark  │
                    │    exists?       │
                    └────┬──────┬─────┘
                         │      │
                    YES  │      │  NO
                         │      │
              ┌──────────▼┐  ┌──▼──────────────┐
              │ Use stored│  │ PCC Classifier   │
              │ mark from │  │ hash → assign    │
              │ conn table│  │ connection-mark  │
              └──────────┬┘  └──┬──────────────┘
                         │      │
                    ┌────▼──────▼─────┐
                    │  routing-mark    │
                    │  set from conn   │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
        ┌─────▼─────┐  ┌────▼────┐  ┌──────▼──────┐
        │ to-WAN1  │  │ to-WAN2 │  │  to-WAN3   │
        │ table    │  │ table   │  │  table     │
        └─────┬─────┘  └────┬────┘  └──────┬──────┘
              │              │              │
        ┌─────▼─────┐  ┌────▼────┐  ┌──────▼──────┐
        │ GW .113.1 │  │GW .100.1│  │  GW .2.1   │
        │ check-gw  │  │check-gw │  │  check-gw  │
        │ ACTIVE?   │  │ACTIVE?  │  │  ACTIVE?   │
        └─────┬─────┘  └────┬────┘  └──────┬──────┘
              │              │              │
         YES  │         YES  │         YES  │
              ▼              ▼              ▼
         [ether1]       [ether2]       [ether3]
              │              │              │
         NO → DROP    NO → DROP     NO → DROP
```

---

## ۴.۵ توضیح جریان NAT

### معماری NAT به‌ازای هر WAN

```
Connection marked WAN1-conn:
  → Route via to-WAN1 → egress ether1
  → NAT: action=masquerade out-interface=ether1
  → Source: 192.168.1.x → 203.0.113.2

Connection marked WAN2-conn:
  → Route via to-WAN2 → egress ether2
  → NAT: action=masquerade out-interface=ether2
  → Source: 192.168.1.x → 198.51.100.2

Connection marked WAN3-conn:
  → Route via to-WAN3 → egress ether3
  → NAT: action=masquerade out-interface=ether3
  → Source: 192.168.1.x → 192.0.2.2
```

### اتصال NAT + PCC

| گام | عمل | نتیجه |
|-----|-----|-------|
| ۱ | PCC تخصیص WAN2-conn | Connection به ISP-2 متصل شد |
| ۲ | Routing از to-WAN2 استفاده می‌کند | بسته از ether2 خارج می‌شود |
| ۳ | NAT masquerade روی ether2 | IP عمومی = 198.51.100.2 |
| ۴ | بسته بازگشتی به 198.51.100.2 | تطابق جدول Connection |
| ۵ | NAT معکوس | مقصد = 192.168.1.x |
| ۶ | Route به LAN | تحویل به کلاینت |

---

## ۴.۶ سفر بسته — مثال کامل

**درخواست:** کاربر `192.168.1.50` باز می‌کند `https://google.com`

| گام | مکان | عمل | جزئیات |
|-----|------|-----|--------|
| ۱ | Client | TCP SYN ارسال | src=192.168.1.50:49821 dst=142.250.80.46:443 |
| ۲ | ether5 | فریم دریافت شد | Destination MAC = router LAN MAC |
| ۳ | Filter forward | تطابق قانون | `in-interface-list=LAN action=accept` |
| ۴ | Mangle prerouting | طبقه‌بندی PCC | hash(50+142.250.80.46+49821+443) mod 3 = 1 → WAN2-conn |
| ۵ | Mangle prerouting | Routing mark | connection-mark=WAN2-conn → routing-mark=to-WAN2 |
| ۶ | Conntrack | ورودی جدید | state=new, mark=WAN2-conn |
| ۷ | Routing | Lookup جدول | to-WAN2: 0.0.0.0/0 via 198.51.100.1 ✓ ACTIVE |
| ۸ | NAT srcnat | Masquerade | 192.168.1.50:49821 → 198.51.100.2:49821 |
| ۹ | ether2 | ارسال | به Gateway ISP-2 |
| ۱۰ | ISP-2 → Internet | Routing | به سرور Google |
| ۱۱ | Internet → ISP-2 | بازگشت SYN-ACK | dst=198.51.100.2:49821 |
| ۱۲ | ether2 | دریافت | جدول Connection: ESTABLISHED, WAN2-conn |
| ۱۳ | NAT reverse | De-translate | dst=192.168.1.50:49821 |
| ۱۴ | Routing | Connected route | 192.168.1.0/24 via ether5 |
| ۱۵ | ether5 | تحویل | به MAC کلاینت |
| ۱۶ | Client | TCP established | Session HTTPS فعال از طریق ISP-2 |

---

## ۴.۷ سناریوی Failover در این طراحی

```
NORMAL STATE:
  ISP-1: ACTIVE (distance=1, PCC bucket 0)
  ISP-2: ACTIVE (distance=1, PCC bucket 1)
  ISP-3: ACTIVE (distance=1, PCC bucket 2)
  Traffic distributed 33/33/33 via PCC

ISP-1 FAILS:
  check-gateway: 203.0.113.1 UNREACHABLE
  to-WAN1 route: INACTIVE
  PCC bucket 0 connections: DROPPED
  New connections: distributed 50/50 between ISP-2 and ISP-3
  Netwatch script: disable WAN1 mangle rules (optional)

ISP-1 RECOVERS:
  check-gateway: 203.0.113.1 REACHABLE
  to-WAN1 route: ACTIVE
  New connections: distributed 33/33/33 again
  Existing sessions on ISP-2/3: unaffected
```

---

**فصل بعد ←** [فصل ۵: پیکربندی MikroTik](../05-mikrotik-configuration/README.md)
