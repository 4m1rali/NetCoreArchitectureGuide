# Recursive Routing

> حل دسترسی Gateway — پایه Multi-WAN قابل‌اعتماد.

---

## تعریف مهندسی

**Recursive Routing** فرآیندی است که در آن روتر دسترسی به آدرس Gateway را از طریق Route دیگری حل می‌کند، نه اینکه فرض کند Gateway مستقیماً متصل است. در MikroTik، این کار از ویژگی‌های **scope** و **target-scope** روی Routeها استفاده می‌کند.

بدون Recursive Routing، `check-gateway` نمی‌تواند سلامت Gateway را روی لینک‌های WAN غیرمستقیم (PPPoE، Metro Ethernet، VLAN handoff) تأیید کند.

---

## توضیح Scope و Target-Scope

| ویژگی | مقدار | معنی |
|-------|-------|------|
| `scope` | 10 | Route برای حل دسترسی Gateway استفاده می‌شود |
| `target-scope` | پیش‌فرض (30) | Route برای Forward واقعی بسته استفاده می‌شود |
| `scope` | 30 | مستقیماً متصل — برای Forwarding |

### جریان حل

```
Default route: 0.0.0.0/0 via 203.0.113.1 (target-scope=30)
  → Router must find how to reach 203.0.113.1
  → Host route: 203.0.113.1/32 via ether1 (scope=10)
  → 203.0.113.1 reachable via ether1
  → Default route: ACTIVE
  → check-gateway pings 203.0.113.1 via ether1
```

---

## الگوهای پیکربندی

### WAN با IP ثابت (مستقیماً متصل)

```
/ip route
add dst-address=203.0.113.1/32 gateway=ether1 scope=10
add dst-address=0.0.0.0/0 gateway=203.0.113.1 distance=1 check-gateway=ping
```

### WAN PPPoE (Interface به‌عنوان Gateway)

```
/interface pppoe-client
add name=pppoe-isp1 interface=ether1 user=isp1@provider password=secret

/ip route
add dst-address=0.0.0.0/0 gateway=pppoe-isp1 distance=1 check-gateway=ping
```

### Metro Ethernet چند Hop

```
/ip route
add dst-address=198.51.100.1/32 gateway=10.0.0.1 scope=10
add dst-address=10.0.0.0/24 gateway=ether2
add dst-address=0.0.0.0/0 gateway=198.51.100.1 distance=1 check-gateway=ping
```

### Recursive با چند هدف مانیتورشده

```
/ip route
add dst-address=203.0.113.1/32 gateway=ether1 scope=10
add dst-address=8.8.8.8/32 gateway=203.0.113.1 scope=10 target-scope=10
add dst-address=0.0.0.0/0 gateway=8.8.8.8 distance=1 check-gateway=ping
```

این روش دسترسی اینترنت را مانیتور می‌کند، نه فقط دسترسی L2 Gateway.

---

## موارد استفاده

| سناریو | چرا Recursive |
|--------|--------------|
| WAN PPPoE | Gateway اینترفیس مجازی است، نه IP |
| Metro Ethernet | Gateway پشت زیرشبکه Provider |
| VLAN WAN handoff | Gateway روی زیرشبکه VLAN متفاوت |
| GRE tunnel WAN | Gateway داخل تونل |
| ISP با زیرشبکه /30 | Gateway در Connected route نیست |

---

## مزایا

| مزیت | جزئیات |
|------|--------|
| check-gateway قابل‌اعتماد | روی هر نوع WAN handoff کار می‌کند |
| مانیتورینگ سطح اینترنت | می‌توان 8.8.8.8 را به‌جای فقط Gateway مانیتور کرد |
| توپولوژی انعطاف‌پذیر | طراحی‌های پیچیده ISP handoff را پشتیبانی می‌کند |
| دقت Failover | Routeهای مرده وقتی Gateway واقعاً غیرقابل‌دسترس است حذف می‌شوند |

---

## معایب و ریسک‌ها

| ریسک | جزئیات |
|------|--------|
| scope اشتباه | Route هرگز Active نمی‌شود |
| حل دایره‌ای | Route Gateway به خودش اشاره می‌کند |
| Routeهای اضافی | پیکربندی بیشتر از مستقیماً متصل |
| پیچیدگی Debug | برای تشخیص به route print detail نیاز است |

---

## دستورات تشخیصی

```
/ip route print detail where dst-address=0.0.0.0/0
/ip route print detail where dst-address~"203.0.113"
/ip route check 203.0.113.1
```

---

**بعدی ←** [Connection Tracking Deep Dive](connection-tracking-deep.md)
