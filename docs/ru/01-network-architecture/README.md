# Глава 1 — Network Architecture

> Глубокая инженерная основа для Multi-WAN развёртываний на MikroTik.

---

## 1.1 Что такое Multi-WAN Architecture?

**Multi-WAN Architecture** — это архитектурный паттерн, при котором один пограничный маршрутизатор одновременно поддерживает подключения к двум или более независимым upstream-провайдерам (ISP) и интеллектуально распределяет трафик между ними или выполняет failover.

### Enterprise-определение

На уровне ISP и Enterprise Multi-WAN — это не просто «два интернет-канала». Это **контролируемая система выбора исходящего пути (egress path selection)**, которая управляет:

- **Path diversity** — несколько независимых маршрутов в интернет
- **Bandwidth aggregation** — использование суммарной ёмкости всех WAN-каналов
- **Resilience** — автоматический failover при деградации или отказе пути
- **Policy control** — направление определённых типов трафика на конкретные WAN-пути

### Уровни архитектуры

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

## 1.2 Роль NAT в реальных сетях

**Network Address Translation (NAT)** — механизм, который сопоставляет частные внутренние адреса с публичными WAN-адресами. В Multi-WAN средах NAT не является опциональным — это связующий слой между внутренними сессиями и внешними путями.

### Почему NAT критичен в Multi-WAN

| Функция | Без NAT | С NAT |
|---------|---------|-------|
| Выход частных IP в интернет | Невозможен в публичной сети | Включён через masquerade/src-nat |
| Согласованность обратного пути | Не определена | Connection tracking обеспечивает симметричную маршрутизацию |
| Маппинг адресов per-WAN | Невозможен | Каждый WAN может иметь свой пул публичных IP |
| Session stickiness | Нарушена | NAT + connection mark = стабильные сессии |

### Типы NAT в Multi-WAN MikroTik

| Тип | Команда | Сценарий использования |
|-----|---------|------------------------|
| Masquerade | `action=masquerade` | Динамический публичный IP от ISP (наиболее распространён) |
| Src-NAT | `action=src-nat to-addresses=X` | Фиксированный публичный IP на каждый WAN |
| Dst-NAT | `action=dst-nat` | Port forwarding с конкретного WAN |

### Поток NAT в Multi-WAN

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

**Критическое правило:** обратный трафик ОБЯЗАН входить через тот же WAN-интерфейс, через который был отправлен исходящий пакет. Connection tracking обеспечивает это.

---

## 1.3 Как таблица маршрутизации принимает решения

Таблица маршрутизации MikroTik — это decision engine для каждого пакета. В Multi-WAN вы обычно работаете с **несколькими таблицами маршрутизации** помимо стандартной таблицы `main`.

### Процесс принятия маршрутного решения

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

### Таблицы маршрутизации в Multi-WAN

| Имя таблицы | Назначение | Создаётся |
|-------------|------------|-----------|
| `main` | Маршруты по умолчанию, failover gateway | Static routes, DHCP |
| `to-WAN1` | PCC-трафик для ISP-1 | Mangle + routing rules |
| `to-WAN2` | PCC-трафик для ISP-2 | Mangle + routing rules |
| `to-WAN3` | PCC-трафик для ISP-3 | Mangle + routing rules |

### Приоритет выбора маршрута

1. **Routing mark** (наивысший приоритет — переопределяет main table)
2. **Scope и target-scope** (рекурсивное разрешение маршрутизации)
3. **Distance** (административная дистанция — меньше = предпочтительнее)
4. **Gateway reachability** (check-gateway ping/interface)

---

## 1.4 Connection Tracking — почему это критично

**Connection Tracking (conntrack)** — stateful inspection engine MikroTik. Он поддерживает таблицу всех активных соединений и их метаданных.

### Что хранит Connection Tracking

| Поле | Назначение |
|------|------------|
| `src-address` / `dst-address` | Исходные endpoints |
| `src-address` / `dst-address` (after NAT) | Транслированные endpoints |
| `connection-mark` | PCC routing mark |
| `routing-mark` | Активный селектор таблицы маршрутизации |
| `connection-state` | new / established / related / invalid |
| `timeout` | Время жизни сессии |
| `protocol` | TCP / UDP / ICMP |

### Почему Connection Tracking — основа Multi-WAN

Без connection tracking:

- **PCC не работает** — marks не сохраняются между пакетами в сессии
- **NAT ломается** — обратный трафик не может быть корректно de-translated
- **Возникает asymmetric routing** — исходящий через WAN1, обратный через WAN2
- **Stateful firewall rules не работают** — `connection-state=established` становится ненадёжным

### Поток Connection Tracking

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

## 1.5 L2 vs L3 vs L4 в контексте Multi-WAN

Понимание того, на каком уровне OSI работает каждый Multi-WAN метод, необходимо для корректного проектирования.

### Layer 2 (Data Link)

| Элемент | Роль в Multi-WAN |
|---------|------------------|
| Ethernet interfaces | Физическое подключение WAN/LAN |
| VLANs | Разделение ISP handoff и внутренней сети |
| Bridge | LAN switching (не routing) |
| PPPoE | Аутентификация ISP и L2-туннель |

**L2 НЕ принимает маршрутных решений.** Он доставляет frames в L3 engine роутера.

### Layer 3 (Network)

| Элемент | Роль в Multi-WAN |
|---------|------------------|
| IP Routing | Выбор пути к destinations |
| ECMP | Equal-cost multipath в таблице маршрутизации |
| Static routes | Default gateway на каждый WAN |
| Recursive routing | Разрешение доступности шлюза |
| Routing marks | Направление трафика в конкретные таблицы |
| NAT | Трансляция адресов |

**L3 — уровень, где принимаются все Multi-WAN маршрутные решения.**

### Layer 4 (Transport)

| Элемент | Роль в Multi-WAN |
|---------|------------------|
| Connection Tracking | Сохранение состояния сессии |
| PCC (Mangle) | Per-connection классификация по src+dst IP+port |
| Firewall | Stateful фильтрация по connection |
| NAT port mapping | Трансляция TCP/UDP портов |

**L4 обеспечивает session awareness, благодаря которому возможны PCC и stateful NAT.**

### Диаграмма взаимодействия уровней

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

## 1.6 Реальный путь пакета через роутер

### Полный пошаговый поток

**Сценарий:** клиент `192.168.1.100` запрашивает `https://example.com` (203.0.113.50) через 3-WAN PCC setup.

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

## 1.7 Архитектурные принципы

| Принцип | Описание |
|---------|----------|
| **Symmetric routing** | Исходящий и обратный трафик должны использовать один WAN-путь |
| **Connection stickiness** | После классификации сессия остаётся на том же пути |
| **Mark before route** | Mangle (PCC) должен выполняться до routing decision |
| **NAT after route mark** | NAT rules должны ссылаться на out-interface или connection-mark |
| **Monitor all gateways** | Каждый WAN gateway должен иметь включённый check-gateway |
| **Separate routing tables** | Одна таблица на WAN для PCC; main table для failover |

---

## 1.8 VRF и изоляция таблиц маршрутизации

**VRF (Virtual Routing and Forwarding)** изолирует таблицы маршрутизации, чтобы разные traffic domains никогда не «утекали» маршруты друг в друга. В RouterOS 7 VRF реализуется через `/routing table` с флагом `fib` и опциональной привязкой VRF interface.

### Когда VRF важен в Multi-WAN

| Сценарий | Использование VRF |
|----------|-------------------|
| ISP wholesale + retail на одном роутере | Разделение customer routing и management |
| MPLS + Internet на одном CCR | Internet Multi-WAN в VRF-A, MPLS в VRF-B |
| Guest + Corporate network | Guest traffic принудительно через filtered WAN |
| Management plane isolation | Out-of-band management никогда не использует customer WAN tables |

### VRF + PCC Architecture

```
VRF: internet-vrf
  ├── ether1 (ISP-1) → to-WAN1 table
  ├── ether2 (ISP-2) → to-WAN2 table
  ├── ether3 (ISP-3) → to-WAN3 table
  └── ether5 (LAN)   → PCC mangle applies here

VRF: mgmt-vrf
  └── ether6 (Management) → single dedicated WAN or VPN
```

### Ключевое правило

PCC mangle, routing tables и NAT должны существовать **внутри одного VRF/FIB context**. Connection mark в одном VRF никогда не должен ссылаться на маршруты в другом.

---

## 1.9 MTU, MSS и фрагментация в Multi-WAN

Разные ISP часто имеют разные значения MTU. Это вызывает silent failures в Multi-WAN deployments.

| Тип ISP | Typical MTU | Примечания |
|---------|-------------|------------|
| Fiber / Ethernet | 1500 | Standard |
| PPPoE | 1492 | 8-byte PPPoE header |
| GRE / VPN overlay | 1400–1476 | Depends on encapsulation |
| LTE | 1400–1428 | Carrier-dependent |

### MSS Clamping (обязательно для PPPoE/LTE WAN)

```
/ip firewall mangle
add chain=forward protocol=tcp tcp-flags=syn action=change-mss new-mss=1440 passthrough=yes \
    out-interface-list=WAN comment="MSS clamp WAN"
```

### Симптомы несоответствия MTU

| Симптом | Причина |
|---------|---------|
| Small packets work, large downloads fail | MTU black hole |
| HTTPS fails, ping works | PMTUD blocked, MSS not clamped |
| One WAN works, another fails same site | Different ISP MTU |
| VPN connects but no traffic | Tunnel MTU too high |

### Per-Interface MTU Setting

```
/interface ethernet
set ether1 mtu=1500
set ether2 mtu=1492 comment="PPPoE ISP-2"
set lte1 mtu=1420 comment="LTE ISP-3"
```

---

## 1.10 Bridge Mode vs Router Mode на WAN

| Mode | Use | Multi-WAN Impact |
|------|-----|-----------------|
| **Router mode** (recommended) | Each WAN is a routed interface | Full PCC, NAT, firewall control |
| **Bridge mode** | WAN bridged to LAN (transparent) | Cannot route between WANs — avoid |
| **Hybrid** | WAN routed, LAN bridged | Most common production pattern |

### Recommended LAN Design

```
/interface bridge
add name=bridge-lan vlan-filtering=no

/interface bridge port
add bridge=bridge-lan interface=ether5
add bridge=bridge-lan interface=ether6

/interface list member
add interface=bridge-lan list=LAN
```

WAN interfaces (ether1–3) **никогда** не должны быть bridge ports в Multi-WAN designs.

---

## 1.11 FastTrack vs Connection Tracking в Multi-WAN

**FastTrack** обходит connection tracking и mangle для производительности. В PCC deployments FastTrack **ломает** load balancing.

| Setting | PCC Compatible | Performance |
|---------|---------------|-------------|
| FastTrack enabled | **NO** | Highest throughput |
| FastTrack disabled | **YES** | Required for PCC |
| FastTrack + HW offload | **NO** | Bypasses mangle entirely |

### Rule Order Impact

```
# WRONG — FastTrack before PCC awareness
add chain=forward action=fasttrack-connection connection-state=established,related

# CORRECT for PCC — no FastTrack on balanced traffic
add chain=forward action=accept connection-state=established,related
```

For CCR hardware at scale without PCC (failover-only), FastTrack is acceptable.

---

## 1.12 Connection State Machine

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

| State | Multi-WAN Behavior |
|-------|-------------------|
| `new` | PCC classifier runs — connection assigned to WAN |
| `established` | Mark read from table — no re-classification |
| `related` | FTP/data channel follows parent connection mark |
| `invalid` | Drop — often indicates asymmetric routing or NAT failure |
| `untracked` | Bypasses conntrack — policy routing must handle manually |

---

## 1.13 ISP Handoff Types

| Handoff | Configuration | Multi-WAN Notes |
|---------|--------------|-----------------|
| Static IP | `/ip address` + static route | Simplest — reference design |
| DHCP | `/ip dhcp-client` | Gateway may change — use script to update routes |
| PPPoE | `/interface pppoe-client` | Use interface as gateway, MTU 1492 |
| BGP | `/routing bgp` | Datacenter/ISP — see Chapter 13 |
| LTE | `/interface lte` | Dynamic IP — monitor with Netwatch |
| VLAN from ISP | `/interface vlan` | Tag per ISP on single physical port |

---

**Следующая глава →** [Глава 2: Core Concepts](../02-core-concepts/README.md)
