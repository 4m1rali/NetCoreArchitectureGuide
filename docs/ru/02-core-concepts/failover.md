# Failover

> Автоматическое восстановление WAN-пути и резервирование шлюзов.

---

## Инженерное определение

**Failover** — механизм сетевой отказоустойчивости, который мониторит состояние WAN gateways и автоматически перенаправляет трафик на резервный путь, когда primary gateway становится недоступным. В MikroTik failover реализуется через **gateway monitoring (check-gateway)** в сочетании с **recursive routing** и **route distance prioritization**.

Failover работает на **Layer 3** и независим от load balancing — он обеспечивает непрерывность, а не распределение.

---

## Внутренний поток роутера (пошагово)

```
1. Static route configured with check-gateway=ping
   → Route: 0.0.0.0/0 via 203.0.113.1 distance=1 check-gateway=ping
2. Router sends ICMP ping to 203.0.113.1 every check-interval
3. Gateway responds → route status: ACTIVE
4. Traffic flows through ISP-1 normally
5. ISP-1 link fails (cable cut, ISP outage, gateway down)
6. Ping timeout → route status: INACTIVE
7. Router selects next best route:
   → 0.0.0.0/0 via 198.51.100.1 distance=2 check-gateway=ping
8. NEW connections use ISP-2
9. EXISTING connections on ISP-1: DROPPED (expected behavior)
10. ISP-1 recovers → ping success → route ACTIVE again
11. New connections can use ISP-1 (if distance=1 is preferred)
```

---

## Поведение в MikroTik RouterOS

### Базовая конфигурация Failover

```
/ip route
add dst-address=0.0.0.0/0 gateway=203.0.113.1 distance=1 check-gateway=ping
add dst-address=0.0.0.0/0 gateway=198.51.100.1 distance=2 check-gateway=ping
add dst-address=0.0.0.0/0 gateway=192.0.2.1 distance=3 check-gateway=ping
```

### Опции check-gateway

| Опция | Поведение |
|-------|-----------|
| `ping` | ICMP ping к gateway IP — наиболее распространён |
| `arp` | ARP resolution check — для directly connected gateways |
| `none` | Без мониторинга — маршрут всегда active (не рекомендуется) |

### Recursive Routing для Failover

Когда gateway не достижим напрямую (например, PPPoE или metro Ethernet):

```
/ip route
add dst-address=203.0.113.1/32 gateway=ether1 scope=10
add dst-address=0.0.0.0/0 gateway=203.0.113.1 distance=1 check-gateway=ping
```

**Поток:**
1. Route to 203.0.113.1 resolved via ether1 (scope=10)
2. Default route uses 203.0.113.1 as gateway
3. check-gateway pings 203.0.113.1
4. If unreachable, both routes become inactive

### Failover с PCC

В production failover работает **наряду** с PCC:

- PCC routes в таблице `to-WAN1` имеют check-gateway на ISP-1 gateway
- Когда ISP-1 fails, маршрут `to-WAN1` table становится inactive
- PCC-classified traffic для WAN1 не имеет active route → **dropped**
- **Решение:** добавить fallback routes или использовать Netwatch scripts для удаления PCC marks для failed WAN

### Netwatch Advanced Failover

```
/tool netwatch
add host=203.0.113.1 interval=10s timeout=3s \
    up-script="/ip route enable [find gateway=203.0.113.1]" \
    down-script="/ip route disable [find gateway=203.0.113.1]"
```

---

## Сценарии использования

| Среда | Применение |
|-------|------------|
| Enterprise HQ | Primary fiber + backup LTE failover |
| ISP Edge | Upstream provider redundancy |
| Branch Office | Dual ISP с автоматическим switchover |
| Critical Services | VoIP gateway не должен терять связность |
| Любой Multi-WAN | Обязательный companion к PCC или ECMP |

---

## Преимущества

| Преимущество | Детали |
|--------------|--------|
| Automatic recovery | Ручное вмешательство не требуется |
| Simple configuration | Distance + check-gateway достаточно |
| Fast detection | Ping timeout обычно 3–10 секунд |
| Works with any method | Совместим с PCC, ECMP или standalone |
| Proven reliability | Стандартный подход у всех router vendors |

---

## Недостатки и риски

| Риск | Детали |
|------|--------|
| Active session loss | Существующие соединения на failed WAN сбрасываются |
| Flapping | Нестабильный link вызывает повторяющийся failover/failback |
| DNS cache issues | Клиенты могут кешировать DNS responses failed WAN |
| PCC orphan connections | Connections marked для dead WAN не имеют route |
| Asymmetric routing on recovery | Обратный трафик может прийти до восстановления маршрута |
| Single point of monitoring | Ping к gateway не обнаруживает ISP-side failures |

### Обнаружение ISP-Side Failures

Gateway ping подтверждает только L2/L3 reachability к ISP gateway — не connectivity в интернет за ним.

**Решение:** мониторить внешний host:

```
/tool netwatch
add host=8.8.8.8 interval=10s timeout=3s \
    up-script="..." down-script="..."
```

Или использовать recursive routing с несколькими monitor targets.

---

## Типичные ошибки реализации

| Ошибка | Последствие | Исправление |
|--------|-------------|-------------|
| Нет check-gateway | Мёртвые маршруты остаются active навсегда | Всегда включать check-gateway=ping |
| Одинаковый distance на всех маршрутах | Непредсказуемый порядок failover | Использовать distance=1,2,3 для приоритета |
| Отсутствие recursive route | Gateway недоступен, маршрут никогда не активируется | Добавить host route к gateway через interface |
| Failover без PCC fallback | PCC traffic blackholed при WAN failure | Реализовать Netwatch scripts или disable PCC marks |
| Слишком агрессивный ping interval | False positives на congested links | Использовать interval=10s timeout=3s minimum |
| Нет upstream monitoring | Gateway up, но ISP routing broken | Мониторить external IP (8.8.8.8, 1.1.1.1) |

---

**Далее →** [Load Balancing](load-balancing.md)
