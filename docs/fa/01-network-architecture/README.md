# فصل ۱ — معماری شبکه

> پایه مهندسی عمیق برای استقرار Multi-WAN روی MikroTik.

---

## ۱.۱ معماری Multi-WAN چیست؟

**معماری Multi-WAN** یک الگوی طراحی شبکه است که در آن یک روتر لبه به‌طور همزمان به دو یا چند ارائه‌دهنده مستقل (ISP) متصل می‌شود و ترافیک را به‌صورت هوشمند بین آن‌ها توزیع یا Failover می‌کند.

### تعریف Enterprise

در سطح ISP و Enterprise، Multi-WAN صرفاً «دو اتصال اینترنت» نیست. این یک **سیستم کنترل‌شده انتخاب مسیر خروجی (Egress)** است که موارد زیر را مدیریت می‌کند:

- **تنوع مسیر (Path diversity)** — چندین مسیر مستقل به اینترنت
- **تجمیع پهنای باند (Bandwidth aggregation)** — استفاده از ظرفیت ترکیبی تمام لینک‌های WAN
- **تاب‌آوری (Resilience)** — Failover خودکار هنگام افت کیفیت یا قطع مسیر
- **کنترل سیاست (Policy control)** — هدایت انواع خاص ترافیک به مسیرهای WAN مشخص

### لایه‌های معماری

```
┌─────────────────────────────────────────────────────────┐
│                    APPLICATION LAYER                     │
│              (HTTP, DNS, VoIP, VPN, etc.)               │
├─────────────────────────────────────────────────────────┤
│                    TRANSPORT LAYER (L4)                  │
│         TCP/UDP — Connection Tracking lives here         │
├─────────────────────────────────────────────────────────┤
│                    NETWORK LAYER (L3)                    │
│    IP Routing · NAT · Firewall · Mangle · PCC/ECMP      │
├─────────────────────────────────────────────────────────┤
│                   DATA LINK LAYER (L2)                   │
│         Ethernet · VLAN · PPPoE · Bridge                │
├─────────────────────────────────────────────────────────┤
│                    PHYSICAL LAYER (L1)                   │
│              Fiber · Copper · SFP · LTE                  │
└─────────────────────────────────────────────────────────┘
```

---

## ۱.۲ نقش NAT در شبکه‌های واقعی

**Network Address Translation (NAT)** مکانیزمی است که آدرس‌های خصوصی داخلی را به آدرس‌های عمومی WAN نگاشت می‌کند. در محیط‌های Multi-WAN، NAT اختیاری نیست — این لایه اتصال بین Sessionهای داخلی و مسیرهای خارجی است.

### چرا NAT در Multi-WAN حیاتی است

| عملکرد | بدون NAT | با NAT |
|--------|----------|--------|
| خروج IP خصوصی | در اینترنت عمومی ممکن نیست | از طریق masquerade/src-nat فعال می‌شود |
| یکنواختی مسیر بازگشت | تعریف‌نشده | Connection Tracking مسیریابی متقارن را تضمین می‌کند |
| نگاشت آدرس به‌ازای هر WAN | ممکن نیست | هر WAN می‌تواند Pool IP عمومی خود را داشته باشد |
| Session Stickiness | مختل می‌شود | NAT + Connection Mark = Session پایدار |

### انواع NAT در Multi-WAN میکروتیک

| نوع | دستور | کاربرد |
|-----|-------|--------|
| Masquerade | `action=masquerade` | IP عمومی پویا از ISP (رایج‌ترین) |
| Src-NAT | `action=src-nat to-addresses=X` | IP عمومی ثابت به‌ازای هر WAN |
| Dst-NAT | `action=dst-nat` | Port Forwarding از WAN مشخص |

### جریان NAT در Multi-WAN

```
Client (192.168.1.50) → Router LAN
    ↓
Routing decision: WAN2 selected (PCC mark)
    ↓
NAT: 192.168.1.50 → 203.0.113.45 (ISP-2 public IP)
    ↓
Egress via ether2 → ISP-2
    ↓
Return: 203.0.113.45 → 192.168.1.50 (connection tracking table)
```

**قانون حیاتی:** ترافیک بازگشتی **باید** از همان اینترفیس WAN که بسته خروجی را حمل کرده وارد شود. Connection Tracking این را اعمال می‌کند.

---

## ۱.۳ جدول مسیریابی چگونه تصمیم می‌گیرد

جدول مسیریابی MikroTik موتور تصمیم‌گیری برای هر بسته است. در Multi-WAN، معمولاً با **چندین جدول مسیریابی** فراتر از جدول پیش‌فرض `main` کار می‌کنید.

### فرآیند تصمیم‌گیری مسیریابی

```
Packet arrives at router
    ↓
Is there a routing-mark on the packet/connection?
    ├── YES → Use the routing table associated with that mark
    └── NO  → Use main routing table
         ↓
    Find best matching route (longest prefix match)
         ↓
    Multiple equal-cost routes?
         ├── YES → ECMP (hash-based distribution)
         └── NO  → Single next-hop selected
              ↓
    Is gateway reachable? (check-gateway)
         ├── YES → Forward packet
         └── NO  → Mark route as inactive, try next route
```

### جداول مسیریابی در Multi-WAN

| نام جدول | هدف | ایجاد شده توسط |
|----------|-----|----------------|
| `main` | مسیرهای پیش‌فرض، Gateway Failover | Static routes، DHCP |
| `to-WAN1` | ترافیک PCC برای ISP-1 | Mangle + Routing rules |
| `to-WAN2` | ترافیک PCC برای ISP-2 | Mangle + Routing rules |
| `to-WAN3` | ترافیک PCC برای ISP-3 | Mangle + Routing rules |

### اولویت انتخاب مسیر

1. **Routing mark** (بالاترین اولویت — جدول main را Override می‌کند)
2. **Scope و target-scope** (حل Recursive Routing)
3. **Distance** (فاصله اداری — کمتر = ترجیح بیشتر)
4. **دسترسی Gateway** (check-gateway ping/interface)

---

## ۱.۴ Connection Tracking — چرا حیاتی است

**Connection Tracking (conntrack)** موتور بازرسی Stateful میکروتیک است. جدولی از تمام Connectionهای فعال و متادیتای آن‌ها را نگه می‌دارد.

### Connection Tracking چه چیزی ذخیره می‌کند

| فیلد | هدف |
|------|-----|
| `src-address` / `dst-address` | Endpointهای اصلی |
| `src-address` / `dst-address` (after NAT) | Endpointهای ترجمه‌شده |
| `connection-mark` | علامت مسیریابی PCC |
| `routing-mark` | انتخاب‌گر جدول مسیریابی فعال |
| `connection-state` | new / established / related / invalid |
| `timeout` | عمر Session |
| `protocol` | TCP / UDP / ICMP |

### چرا Connection Tracking پایه Multi-WAN است

بدون Connection Tracking:

- **PCC شکست می‌خورد** — Markها بین بسته‌های یک Session پایدار نمی‌مانند
- **NAT مختل می‌شود** — ترافیک بازگشتی نمی‌تواند درست De-translate شود
- **مسیریابی نامتقارن رخ می‌دهد** — خروج از WAN1، بازگشت از WAN2
- **قوانین Stateful Firewall شکست می‌خورند** — `connection-state=established` غیرقابل‌اعتماد می‌شود

### جریان Connection Tracking

```
Packet 1 (SYN) → NEW connection
    → Mangle: assign connection-mark "WAN2-conn"
    → NAT: translate source
    → Route via to-WAN2 table
    → Store in connection table

Packet 2 (ACK) → ESTABLISHED connection
    → Read connection-mark from table (no re-classification)
    → NAT: use stored translation
    → Route via to-WAN2 table (same path)

Packet N (FIN) → Connection teardown
    → Remove from connection table after timeout
```

---

## ۱.۵ L2 در مقابل L3 در مقابل L4 در بستر Multi-WAN

درک اینکه هر روش Multi-WAN در کدام لایه OSI عمل می‌کند برای طراحی صحیح ضروری است.

### لایه ۲ (Data Link)

| عنصر | نقش در Multi-WAN |
|------|------------------|
| اینترفیس‌های Ethernet | اتصال فیزیکی WAN/LAN |
| VLANها | جداسازی Handoff ISP از شبکه داخلی |
| Bridge | سوئیچینگ LAN (نه Routing) |
| PPPoE | احراز هویت ISP و تونل L2 |

**L2 تصمیم مسیریابی نمی‌گیرد.** فریم‌ها را به موتور L3 روتر تحویل می‌دهد.

### لایه ۳ (Network)

| عنصر | نقش در Multi-WAN |
|------|------------------|
| IP Routing | انتخاب مسیر به مقصد |
| ECMP | Equal-Cost Multipath در جدول مسیریابی |
| Static routes | Default Gateway به‌ازای هر WAN |
| Recursive routing | حل دسترسی Gateway |
| Routing marks | هدایت ترافیک به جداول مشخص |
| NAT | ترجمه آدرس |

**L3 جایی است که تمام تصمیمات مسیریابی Multi-WAN اتفاق می‌افتد.**

### لایه ۴ (Transport)

| عنصر | نقش در Multi-WAN |
|------|------------------|
| Connection Tracking | پایداری وضعیت Session |
| PCC (Mangle) | طبقه‌بندی به‌ازای Connection با src+dst IP+port |
| Firewall | فیلتر Stateful بر اساس Connection |
| NAT port mapping | ترجمه پورت TCP/UDP |

**L4 آگاهی Session را فراهم می‌کند که PCC و NAT Stateful را ممکن می‌سازد.**

### دیاگرام تعامل لایه‌ها

```
[L2: Frame arrives on ether1]
        ↓
[L3: IP header processed, routing table consulted]
        ↓
[L4: Connection tracking lookup/creation]
        ↓
[L4: Mangle applies connection-mark (PCC)]
        ↓
[L3: Routing mark set, specific table selected]
        ↓
[L3/L4: NAT translation applied]
        ↓
[L3: Forwarding decision → egress interface]
        ↓
[L2: Frame transmitted on WAN interface]
```

---

## ۱.۶ سفر واقعی بسته از میان روتر

### جریان گام‌به‌گام کامل

**سناریو:** کلاینت `192.168.1.100` درخواست `https://example.com` (203.0.113.50) را از طریق راه‌اندازی PCC سه‌WAN ارسال می‌کند.

```
STEP 1 — INGRESS (LAN → Router)
─────────────────────────────────
Interface: ether5 (LAN bridge)
Source: 192.168.1.100:52431
Destination: 203.0.113.50:443
Protocol: TCP SYN

STEP 2 — FIREWALL: INPUT CHAIN
─────────────────────────────────
Rule: accept established,related → SKIP (new connection)
Rule: accept from LAN → PASS

STEP 3 — FIREWALL: FORWARD CHAIN
─────────────────────────────────
Rule: accept established,related → SKIP (new)
Rule: fasttrack → SKIP (new connection, no fasttrack yet)
Rule: accept from LAN to WAN → PASS

STEP 4 — MANGLE: PREROUTING
─────────────────────────────────
Rule: PCC classifier
  → connection-mark: WAN2-conn (hash of src+dst → ISP-2)
  → routing-mark: to-WAN2

STEP 5 — CONNECTION TRACKING
─────────────────────────────────
Action: CREATE new connection entry
  → Store: src=192.168.1.100:52431, dst=203.0.113.50:443
  → Store: connection-mark=WAN2-conn
  → State: new → established (after handshake)

STEP 6 — ROUTING DECISION
─────────────────────────────────
Routing-mark: to-WAN2 → use table "to-WAN2"
Route: 0.0.0.0/0 via 198.51.100.1 (ISP-2 gateway) → ACTIVE
Next-hop interface: ether2

STEP 7 — NAT: SRCNAT
─────────────────────────────────
Rule: masquerade out-interface=ether2
  → 192.168.1.100:52431 → 198.51.100.45:52431

STEP 8 — FIREWALL: FORWARD (POST-NAT)
─────────────────────────────────
Rule: accept to WAN → PASS

STEP 9 — EGRESS
─────────────────────────────────
Interface: ether2
Source: 198.51.100.45:52431
Destination: 203.0.113.50:443
→ Transmitted to ISP-2

STEP 10 — RETURN PATH
─────────────────────────────────
Interface: ether2 (MUST be same WAN)
Destination: 198.51.100.45:52431
→ Connection table lookup → ESTABLISHED
→ routing-mark: to-WAN2 (from connection table)
→ NAT reverse: 198.51.100.45 → 192.168.1.100
→ Forward to ether5 (LAN)
```

---

## ۱.۷ اصول معماری

| اصل | توضیح |
|-----|-------|
| **مسیریابی متقارن (Symmetric routing)** | ترافیک خروجی و بازگشتی باید از یک مسیر WAN استفاده کند |
| **Connection Stickiness** | پس از طبقه‌بندی Session، روی همان مسیر می‌ماند |
| **Mark قبل از Route** | Mangle (PCC) باید قبل از تصمیم مسیریابی اجرا شود |
| **NAT پس از Route Mark** | قوانین NAT باید به out-interface یا connection-mark ارجاع دهند |
| **مانیتور تمام Gatewayها** | هر Gateway WAN باید check-gateway فعال داشته باشد |
| **جداول مسیریابی مجزا** | یک جدول به‌ازای هر WAN برای PCC؛ جدول main برای Failover |

---

## ۱.۸ VRF و ایزولاسیون جدول مسیریابی

**VRF (Virtual Routing and Forwarding)** جداول مسیریابی را ایزوله می‌کند تا دامنه‌های ترافیک مختلف هرگز Routeها را بین یکدیگر نشت ندهند. در RouterOS 7، VRF از طریق `/routing table` با پرچم `fib` و اتصال اختیاری Interface VRF پیاده‌سازی می‌شود.

### چه زمانی VRF در Multi-WAN مهم است

| سناریو | استفاده VRF |
|--------|-------------|
| ISP wholesale + retail روی یک روتر | جداسازی مسیریابی مشتری از مدیریت |
| MPLS + Internet روی یک CCR | Multi-WAN اینترنت در VRF-A، MPLS در VRF-B |
| Guest + شبکه سازمانی | ترافیک مهمان از WAN فیلترشده عبور می‌کند |
| ایزولاسیون صفحه مدیریت | مدیریت Out-of-band هرگز از جداول WAN مشتری استفاده نمی‌کند |

### معماری VRF + PCC

```
VRF: internet-vrf
  ├── ether1 (ISP-1) → to-WAN1 table
  ├── ether2 (ISP-2) → to-WAN2 table
  ├── ether3 (ISP-3) → to-WAN3 table
  └── ether5 (LAN)   → PCC mangle applies here

VRF: mgmt-vrf
  └── ether6 (Management) → single dedicated WAN or VPN
```

### قانون کلیدی

Mangle PCC، جداول مسیریابی و NAT باید همگی **در همان بستر VRF/FIB** وجود داشته باشند. Connection Mark در یک VRF هرگز نباید به Routeهای VRF دیگر ارجاع دهد.

---

## ۱.۹ MTU، MSS و Fragmentation در Multi-WAN

ISPهای مختلف اغلب مقادیر MTU متفاوتی دارند. این باعث شکست‌های خاموش در استقرارهای Multi-WAN می‌شود.

| نوع ISP | MTU معمول | نکات |
|---------|-----------|------|
| Fiber / Ethernet | 1500 | استاندارد |
| PPPoE | 1492 | هدر 8 بایتی PPPoE |
| GRE / VPN overlay | 1400–1476 | بسته به Encapsulation |
| LTE | 1400–1428 | وابسته به Carrier |

### MSS Clamping (الزامی برای WAN PPPoE/LTE)

```
/ip firewall mangle
add chain=forward protocol=tcp tcp-flags=syn action=change-mss new-mss=1440 passthrough=yes \
    out-interface-list=WAN comment="MSS clamp WAN"
```

### علائم ناسازگاری MTU

| علامت | علت |
|-------|-----|
| بسته‌های کوچک کار می‌کنند، دانلودهای بزرگ شکست می‌خورند | MTU black hole |
| HTTPS شکست می‌خورد، ping کار می‌کند | PMTUD مسدود، MSS Clamp نشده |
| یک WAN کار می‌کند، دیگری همان سایت را نه | MTU متفاوت ISP |
| VPN وصل می‌شود اما ترافیک ندارد | MTU تونل خیلی بالا |

### تنظیم MTU به‌ازای هر Interface

```
/interface ethernet
set ether1 mtu=1500
set ether2 mtu=1492 comment="PPPoE ISP-2"
set lte1 mtu=1420 comment="LTE ISP-3"
```

---

## ۱.۱۰ Bridge Mode در مقابل Router Mode روی WAN

| حالت | استفاده | تأثیر Multi-WAN |
|------|---------|-----------------|
| **Router mode** (پیشنهادی) | هر WAN یک Interface Routed است | کنترل کامل PCC، NAT، Firewall |
| **Bridge mode** | WAN به LAN Bridge شده (شفاف) | نمی‌توان بین WANها Route کرد — اجتناب کنید |
| **Hybrid** | WAN Routed، LAN Bridged | رایج‌ترین الگوی Production |

### طراحی پیشنهادی LAN

```
/interface bridge
add name=bridge-lan vlan-filtering=no

/interface bridge port
add bridge=bridge-lan interface=ether5
add bridge=bridge-lan interface=ether6

/interface list member
add interface=bridge-lan list=LAN
```

اینترفیس‌های WAN (ether1–3) در طراحی‌های Multi-WAN **هرگز** نباید Bridge port باشند.

---

## ۱.۱۱ FastTrack در مقابل Connection Tracking در Multi-WAN

**FastTrack** برای کارایی از Connection Tracking و mangle عبور می‌کند. در استقرارهای PCC، FastTrack **Load Balancing را می‌شکند**.

| تنظیم | سازگار با PCC | کارایی |
|-------|---------------|--------|
| FastTrack فعال | **خیر** | بالاترین Throughput |
| FastTrack غیرفعال | **بله** | الزامی برای PCC |
| FastTrack + HW offload | **خیر** | mangle را کاملاً Bypass می‌کند |

### تأثیر ترتیب قوانین

```
# WRONG — FastTrack before PCC awareness
add chain=forward action=fasttrack-connection connection-state=established,related

# CORRECT for PCC — no FastTrack on balanced traffic
add chain=forward action=accept connection-state=established,related
```

برای CCR در مقیاس بدون PCC (فقط Failover)، FastTrack قابل‌قبول است.

---

## ۱.۱۲ ماشین وضعیت Connection

```
                    ┌──────────┐
         ┌─────────│   NEW    │─────────┐
         │         └────┬─────┘         │
         │              │               │
    (invalid)      (SYN-ACK)         (RST)
         │              │               │
         ▼              ▼               ▼
    ┌─────────┐  ┌─────────────┐  ┌─────────┐
    │ INVALID │  │ ESTABLISHED │  │  DROP   │
    └─────────┘  └──────┬──────┘  └─────────┘
                        │
                   (FIN / timeout)
                        │
                        ▼
                 ┌─────────────┐
                 │  TIME-WAIT  │
                 └─────────────┘
```

| وضعیت | رفتار Multi-WAN |
|-------|-----------------|
| `new` | Classifier PCC اجرا می‌شود — Connection به WAN تخصیص می‌یابد |
| `established` | Mark از جدول خوانده می‌شود — بدون طبقه‌بندی مجدد |
| `related` | کانال داده FTP از Connection Mark والد پیروی می‌کند |
| `invalid` | Drop — اغلب نشان‌دهنده مسیریابی نامتقارن یا شکست NAT |
| `untracked` | از conntrack عبور می‌کند — Policy routing باید دستی مدیریت شود |

---

## ۱.۱۳ انواع ISP Handoff

| Handoff | پیکربندی | نکات Multi-WAN |
|---------|----------|----------------|
| IP ثابت | `/ip address` + static route | ساده‌ترین — طراحی مرجع |
| DHCP | `/ip dhcp-client` | Gateway ممکن است تغییر کند — از اسکریپت برای به‌روزرسانی Route استفاده کنید |
| PPPoE | `/interface pppoe-client` | از Interface به‌عنوان Gateway استفاده کنید، MTU 1492 |
| BGP | `/routing bgp` | Datacenter/ISP — فصل ۱۳ را ببینید |
| LTE | `/interface lte` | IP پویا — با Netwatch مانیتور کنید |
| VLAN از ISP | `/interface vlan` | Tag به‌ازای هر ISP روی یک پورت فیزیکی |

---

**فصل بعد ←** [فصل ۲: مفاهیم پایه](../02-core-concepts/README.md)
