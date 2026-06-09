# پیوست ب — برگه مرجع سریع

---

## PCC 3-WAN (Copy-Paste)

```
/routing table add name=to-WAN1 fib
/routing table add name=to-WAN2 fib
/routing table add name=to-WAN3 fib

/ip route
add dst-address=0.0.0.0/0 gateway=GW1 routing-table=to-WAN1 distance=1 check-gateway=ping
add dst-address=0.0.0.0/0 gateway=GW2 routing-table=to-WAN2 distance=1 check-gateway=ping
add dst-address=0.0.0.0/0 gateway=GW3 routing-table=to-WAN3 distance=1 check-gateway=ping
add dst-address=0.0.0.0/0 gateway=GW1 distance=1 check-gateway=ping
add dst-address=0.0.0.0/0 gateway=GW2 distance=2 check-gateway=ping
add dst-address=0.0.0.0/0 gateway=GW3 distance=3 check-gateway=ping

/ip firewall mangle
add chain=prerouting in-interface-list=LAN connection-mark=no-mark dst-address-type=!local per-connection-classifier=both-addresses-and-ports:3/0 action=mark-connection new-connection-mark=WAN1-conn passthrough=yes
add chain=prerouting in-interface-list=LAN connection-mark=no-mark dst-address-type=!local per-connection-classifier=both-addresses-and-ports:3/1 action=mark-connection new-connection-mark=WAN2-conn passthrough=yes
add chain=prerouting in-interface-list=LAN connection-mark=no-mark dst-address-type=!local per-connection-classifier=both-addresses-and-ports:3/2 action=mark-connection new-connection-mark=WAN3-conn passthrough=yes
add chain=prerouting connection-mark=WAN1-conn action=mark-routing new-routing-mark=to-WAN1 passthrough=yes
add chain=prerouting connection-mark=WAN2-conn action=mark-routing new-routing-mark=to-WAN2 passthrough=yes
add chain=prerouting connection-mark=WAN3-conn action=mark-routing new-routing-mark=to-WAN3 passthrough=yes

/ip firewall nat
add chain=srcnat out-interface=WAN1 action=masquerade
add chain=srcnat out-interface=WAN2 action=masquerade
add chain=srcnat out-interface=WAN3 action=masquerade
```

---

## دستورات Debug یک‌خطی

| وظیفه | دستور |
|-------|-------|
| وضعیت Route | `/ip route print detail` |
| Connectionهای فعال | `/ip firewall connection print count-only` |
| توزیع PCC | `/ip firewall connection print count-only where connection-mark="WAN1-conn"` |
| Hitهای Mangle | `/ip firewall mangle print stats` |
| ترافیک به‌ازای هر WAN | `/interface monitor-traffic ether1,ether2,ether3 once` |
| تست Gateway | `/ping GW1 count=20` |
| تست اینترنت | `/ping 8.8.8.8 interface=ether1 count=20` |
| Connectionهای زنده | `/tool torch interface=bridge-lan` |
| پشتیبان Config | `/export file=backup` |

---

## انتخاب روش

| نیاز | استفاده کنید |
|------|-------------|
| NAT + balance | PCC |
| بدون NAT + balance | ECMP |
| مسیر پشتیبان | Failover |
| VoIP روی بهترین WAN | Policy Routing |
| IP + ASN اختصاصی | BGP |
| < 50 کاربر، ساده | فقط Failover |

---

## راه‌حل‌های رایج

| مشکل | راه‌حل |
|------|--------|
| Sessionها قطع می‌شوند | FastTrack را غیرفعال کنید، PCC نه ECMP |
| یک WAN 0٪ | Route inactive و check-gateway را بررسی کنید |
| IP NAT اشتباه | masquerade به‌ازای هر Interface |
| VoIP قطع می‌شود | Policy route بالای PCC |
| دانلودهای بزرگ شکست می‌خورند | MSS clamp + MTU را بررسی کنید |
| CPU بالا | قوانین mangle را کاهش دهید، سخت‌افزار را ارتقا دهید |

---

## تفاوت‌های RouterOS 6 و 7

| ویژگی | ROS 6 | ROS 7 |
|-------|-------|-------|
| جداول مسیریابی | `/routing mark` + `mangle` | `/routing table` + `routing-table=` |
| BGP | `/routing bgp instance` | `/routing bgp template` + `connection` |
| Firewall | `/ip firewall` | `/ip firewall` + `/ipv6 firewall` |
| ECMP per-conn | موجود نیست | `ecmp-per-connection=yes` |
| VRF | محدود | پشتیبانی کامل VRF |

---

**[← بازگشت به فهرست](../../README.md)**
