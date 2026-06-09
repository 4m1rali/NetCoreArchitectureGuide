# Policy Routing

> انتخاب مسیر WAN آگاه از اپلیکیشن و مقصد.

---

## تعریف مهندسی

**Policy Routing** ترافیک مشخص را بر اساس معیارهایی فراتر از Prefix مقصد — زیرشبکه مبدأ، پروتکل، پورت، کاربر، زمان یا حجم Connection — به مسیرهای WAN مشخص هدایت می‌کند. در MikroTik، Policy Routing از طریق **قوانین mangle** (Connection/Routing Mark)، **Routing rules** و **Address list** پیاده‌سازی می‌شود.

برخلاف PCC (که تمام ترافیک را به‌طور یکنواخت توزیع می‌کند)، Policy Routing **کنترل قطعی مسیر** را فراهم می‌کند.

---

## جریان داخلی روتر

```
1. Packet arrives from LAN
2. Mangle PREROUTING — policy rules evaluated FIRST (before PCC)
   a. Match: src-address=192.168.1.0/28 → mark WAN1-conn (VoIP subnet)
   b. Match: protocol=tcp dst-port=22 → mark WAN3-conn (management)
   c. Match: dst-address-list=streaming → mark WAN2-conn
3. If no policy match → PCC classifier runs (fallback distribution)
4. Routing mark applied → correct routing table selected
5. NAT and forwarding proceed normally
```

**ترتیب قوانین حیاتی است:** قوانین Policy باید **بالای** قوانین Classifier PCC قرار گیرند.

---

## الگوهای پیکربندی MikroTik

### VoIP به WAN با کمترین Latency

```
/ip firewall mangle
add chain=prerouting src-address=192.168.1.240/28 protocol=udp \
    dst-port=5060,5061,10000-20000 action=mark-connection \
    new-connection-mark=WAN1-conn passthrough=yes comment="VoIP → ISP-1"
```

### ترافیک مدیریت به WAN اختصاصی

```
/ip firewall mangle
add chain=prerouting src-address=192.168.1.0/28 protocol=tcp \
    dst-port=22,8291 action=mark-connection \
    new-connection-mark=WAN3-conn passthrough=yes comment="Mgmt → ISP-3"
```

### اجبار کاربر مشخص به WAN مشخص

```
/ip firewall address-list
add list=user-john address=192.168.1.55

/ip firewall mangle
add chain=prerouting src-address-list=user-john action=mark-connection \
    new-connection-mark=WAN2-conn passthrough=yes
```

### Routing Rules (RouterOS 7)

```
/routing rule
add src-address=192.168.1.0/24 action=lookup-only-in-table table=to-WAN1
```

---

## موارد استفاده

| محیط | سیاست |
|------|-------|
| Enterprise | VoIP → فیبر، Bulk → WAN ارزان‌تر |
| ISP | مشتریان Premium → Upstream با Latency کم |
| شعبه | ERP → فقط WAN اصلی |
| Gaming lounge | زیرشبکه Gaming → ISP با کمترین Latency |
| Compliance | ترافیک مالی → مسیر WAN ممیزی‌شده |

---

## مزایا

| مزیت | جزئیات |
|------|--------|
| کنترل دقیق | به‌ازای زیرشبکه، پروتکل، کاربر |
| رعایت SLA | اپلیکیشن‌های حیاتی همیشه روی بهترین مسیر |
| بهینه‌سازی هزینه | Bulk روی لینک ارزان، Realtime روی Premium |
| ترکیب با PCC | ابتدا Policy، سپس PCC بقیه را توزیع می‌کند |
| Audit trail | Address list سیاست ترافیک را مستند می‌کند |

---

## معایب و ریسک‌ها

| ریسک | جزئیات |
|------|--------|
| انفجار قوانین | قوانین زیاد → سربار CPU |
| بار نگهداری | تغییر IP نیاز به به‌روزرسانی لیست دارد |
| سردرگمی Bypass | کاربران روی زیرشبکه اشتباه مسیر اشتباه می‌گیرند |
| تداخل با PCC | Markهای همپوشان مسیریابی غیرقابل‌پیش‌بینی ایجاد می‌کنند |
| بدون تطبیق پویا | قوانین Static به افت WAN واکنش نشان نمی‌دهند |

---

## خطاهای رایج

| خطا | راه‌حل |
|-----|--------|
| قوانین Policy زیر قوانین PCC | قوانین Policy را به بالای mangle منتقل کنید |
| یک Connection دوبار Mark شده | فقط روی PCC از `connection-mark=no-mark` استفاده کنید |
| Policy route بدون تطابق NAT | NAT out-interface را با WAN سیاست هم‌راستا کنید |
| فراموش کردن passthrough=yes | فقط اولین بسته Mark می‌گیرد |

---

**بعدی ←** [Recursive Routing](recursive-routing.md)
