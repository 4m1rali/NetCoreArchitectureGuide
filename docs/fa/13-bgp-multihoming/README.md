# فصل ۱۳ — BGP Multi-Homing

> اتصال اینترنت چند-خانه ISP-grade با BGP روی MikroTik.

---

## ۱۳.۱ چه زمانی BGP جایگزین PCC می‌شود

| سناریو | روش |
|--------|-----|
| SOHO / شعبه (۲–۳ ISP) | PCC + Failover |
| Enterprise با /24+ عمومی | BGP multi-homing |
| ISP با ASN اختصاصی | BGP به چند Upstream |
| Datacenter | BGP + ECMP |

BGP زمانی مناسب است که **فضای IP مستقل از Provider (PI)** و **ASN** داشته باشید و ISPها به BGP peering موافق باشند.

---

## ۱۳.۲ معماری BGP Multi-WAN

```
                    ┌──────────────┐
                    │   ISP-1 AS   │
                    │   65001      │
                    └──────┬───────┘
                           │ eBGP
                    ┌──────▼───────┐
                    │  MikroTik    │
                    │  AS 65050    │
                    │  203.0.113.0/24
                    └──────┬───────┘
                           │ eBGP
                    ┌──────▼───────┐
                    │   ISP-2 AS   │
                    │   65002      │
                    └──────────────┘
```

---

## ۱۳.۳ پیکربندی BGP

```
/routing bgp template
add name=isp-peers as=65050 router-id=203.0.113.2

/routing bgp connection
add name=isp1 remote.address=203.0.113.1 .as=65001 local.role=ebgp \
    templates=isp-peers
add name=isp2 remote.address=198.51.100.1 .as=65002 local.role=ebgp \
    templates=isp-peers

/routing filter rule
add chain=bgp-in rule="if (bgp-ospf-nssa-is-not-supported-yet) " disabled=yes
```

### Advertise کردن Prefix خود

```
/routing bgp connection
set isp1 output.network=bgp-networks
set isp2 output.network=bgp-networks

/routing bgp network
add network=203.0.113.0/24 synchronize=no
```

---

## ۱۳.۴ انتخاب مسیر BGP و Load Balancing

| روش | چگونه |
|-----|-------|
| Primary/Backup | AS-PATH prepend روی ISP پشتیبان |
| Load sharing | Accept default از هر دو، ECMP |
| Partial routes | ISP فقط default route می‌فرستد |
| Full table | ISP جدول کامل اینترنت می‌فرستد (نیاز به RAM) |

### Prepend برای Backup

```
/routing filter rule
add chain=bgp-out rule="set bgp-path prepend 65050,65050,65050" \
    comment="Prepend ISP-2 — make ISP-1 preferred"
```

---

## ۱۳.۴ مقایسه BGP در مقابل PCC

| ویژگی | BGP | PCC |
|-------|-----|-----|
| مالکیت IP | نیاز به PI space | هر IP (با NAT) |
| همکاری ISP | نیاز به BGP session | فقط اتصال اینترنت |
| کنترل ورودی | بله (انتخاب مسیر) | خیر (فقط DNS) |
| پیچیدگی | بسیار زیاد | متوسط |
| Failover | خودکار (route withdrawal) | check-gateway |
| هزینه | ASN + هزینه بلوک IP | هزینه استاندارد ISP |

---

## ۱۳.۵ مانیتورینگ BGP

```
/routing bgp session print
/routing bgp advertisement print
/routing route print where bgp=yes
/tool traceroute address=8.8.8.8
```

---

**فصل بعد ←** [فصل ۱۴: مانیتورینگ و عملیات](../14-monitoring-operations/README.md)
