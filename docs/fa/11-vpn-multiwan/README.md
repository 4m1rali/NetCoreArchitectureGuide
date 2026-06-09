# فصل ۱۱ — VPN روی Multi-WAN

> استقرار IPsec، WireGuard و L2TP در چند مسیر WAN.

---

## ۱۱.۱ VPN و Multi-WAN — قانون اصلی

**یک تونل VPN یک Connection واحد است — نمی‌توان آن را با PCC بین WANها متعادل کرد.**

هر تونل VPN باید به **یک Interface WAN مشخص** متصل شود. از Policy Routing برای انتخاب WAN استفاده کنید و برای تاب‌آوری تونل‌های افزون روی WANهای مختلف مستقر کنید.

---

## ۱۱.۲ الگوی معماری

```
                    ┌──────────────┐
                    │  VPN Server  │
                    │  (HQ/DMZ)    │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
        IPsec WAN1   WireGuard WAN2   L2TP WAN3
        (Primary)    (Backup)         (Mobile)
              │            │            │
              └────────────┼────────────┘
                           │
                    ┌──────▼───────┐
                    │   MikroTik   │
                    │  Branch CCR  │
                    └──────────────┘
```

---

## ۱۱.۳ IPsec روی WAN مشخص

```
/ip ipsec peer
add name=peer-hq address=203.0.113.100 local-address=203.0.113.2 \
    exchange-mode=ike2

/ip ipsec identity
add peer=peer-hq auth-method=pre-shared-key secret=strong-preshared-key

/ip ipsec policy
add src-address=192.168.1.0/24 dst-address=10.0.0.0/24 peer=peer-hq \
    tunnel=yes action=encrypt

/ip firewall mangle
add chain=prerouting dst-address=203.0.113.100 action=mark-routing \
    new-routing-mark=to-WAN1 passthrough=yes comment="IPsec via ISP-1"
```

---

## ۱۱.۴ WireGuard روی WAN مشخص

```
/interface wireguard
add name=wg-hq listen-port=51820 mtu=1420

/interface wireguard peers
add interface=wg-hq public-key="HQ_PUBLIC_KEY" endpoint-address=198.51.100.100 \
    endpoint-port=51820 allowed-address=10.0.0.0/24

/ip address
add address=10.0.0.2/24 interface=wg-hq

/ip firewall mangle
add chain=prerouting out-interface=wg-hq action=mark-routing \
    new-routing-mark=to-WAN2 passthrough=yes comment="WireGuard via ISP-2"
```

---

## ۱۱.۵ Failover VPN بین WANها

دو تونل به همان Endpoint HQ از WANهای مختلف مستقر کنید:

```
# Primary: IPsec via WAN1
# Backup: IPsec via WAN2 (disabled until WAN1 fails)

/tool netwatch
add host=203.0.113.1 interval=10s timeout=3s \
    down-script={
        /interface wireguard enable wg-hq-backup
        /ip route enable [find comment="VPN-backup-route"]
    } \
    up-script={
        /interface wireguard disable wg-hq-backup
        /ip route disable [find comment="VPN-backup-route"]
    }
```

---

## ۱۱.۶ دسترسی از راه دور (Road Warrior) روی Multi-WAN

VPN را روی تمام IPهای عمومی WAN منتشر کنید:

```
# L2TP/IPsec on all WAN interfaces
/interface l2tp-server server
set enabled=yes max-mtu=1450 max-mru=1450

/ip firewall nat
add chain=dstnat in-interface=ether1 protocol=udp dst-port=500,4500,1701 action=accept
add chain=dstnat in-interface=ether2 protocol=udp dst-port=500,4500,1701 action=accept
```

کلاینت‌ها به هر IP عمومی قابل‌دسترس متصل می‌شوند.

---

## ۱۱.۷ تنظیم کارایی VPN

| تنظیم | مقدار | دلیل |
|-------|-------|------|
| MTU | 1400–1420 | جلوگیری از Fragmentation روی WAN PPPoE/LTE |
| MSS clamp | 1360 | سربار هدر VPN |
| keepalive | 10s | تشخیص سریع تونل مرده |
| FastTrack | غیرفعال روی ترافیک VPN | حفظ Markهای mangle |

```
/ip firewall mangle
add chain=forward out-interface=wg-hq protocol=tcp tcp-flags=syn \
    action=change-mss new-mss=1360 passthrough=yes
```

---

## ۱۱.۸ خطاهای رایج VPN + Multi-WAN

| خطا | پیامد | راه‌حل |
|-----|-------|--------|
| VPN بدون Policy route | ترافیک تونل PCC-balanced → قطع می‌شود | routing-mark را به WAN مشخص اجبار کنید |
| NAT روی IPsec | Double NAT IKE را می‌شکند | IPsec را قبل از masquerade Accept کنید |
| همان تونل روی دو WAN | تداخل IKE rekey | یک تونل به‌ازای هر WAN |
| بدون MSS clamp | شکست متناوب بسته‌های بزرگ | MSS را روی Interface VPN Clamp کنید |

---

**فصل بعد ←** [فصل ۱۲: IPv6 Multi-WAN](../12-ipv6-multiwan/README.md)
