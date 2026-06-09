# ECMP Routing

> Equal-Cost Multi-Path — распределение нагрузки на уровне L3 маршрутизации.

---

## Инженерное определение

**ECMP (Equal-Cost Multi-Path)** — механизм маршрутизации, при котором несколько next-hop путей к одному и тому же destination prefix имеют идентичные metric (distance) значения, что заставляет роутер одновременно распределять трафик по всем доступным путям.

На enterprise-уровне ECMP работает исключительно на **Layer 3** — он хеширует каждый пакет (или flow, в зависимости от реализации) на один из нескольких equal-cost gateways без session awareness.

---

## Внутренний поток роутера (пошагово)

```
1. Packet arrives with destination 0.0.0.0/0 (default route)
2. Routing table lookup in "main" table
3. Multiple routes found with SAME distance:
   → 0.0.0.0/0 via 203.0.113.1 distance=1
   → 0.0.0.0/0 via 198.51.100.1 distance=1
   → 0.0.0.0/0 via 192.0.2.1 distance=1
4. ECMP hash algorithm applied:
   → Input: src-IP, dst-IP, protocol, src-port, dst-port
   → Output: selected gateway index (0, 1, or 2)
5. Packet forwarded via selected gateway interface
6. NEXT packet in same TCP session MAY use different gateway
   (no connection tracking involvement in pure ECMP)
```

---

## Поведение в MikroTik RouterOS

### Шаблон конфигурации

```
/ip route
add dst-address=0.0.0.0/0 gateway=203.0.113.1 distance=1 check-gateway=ping
add dst-address=0.0.0.0/0 gateway=198.51.100.1 distance=1 check-gateway=ping
add dst-address=0.0.0.0/0 gateway=192.0.2.1 distance=1 check-gateway=ping
```

### Специфика MikroTik

| Поведение | Детали |
|-----------|--------|
| Hash input | Поля заголовков Layer 3/4 |
| Distribution model | Per-packet (по умолчанию) или per-connection (с ECMP per-connection-setting) |
| Route removal | При сбое check-gateway маршрут становится inactive — трафик перераспределяется |
| Table scope | Работает только в одной таблице маршрутизации |
| NAT interaction | Проблематично — сессии могут нарушаться из-за per-packet смены пути |

### ECMP Per-Connection Mode (RouterOS 7+)

```
/routing settings
set ecmp-per-connection=yes
```

Это делает ECMP более похожим на flow-based распределение, улучшая совместимость с NAT.

---

## Сценарии использования

| Среда | Применение |
|-------|------------|
| ISP Core | Несколько upstream peer links с BGP |
| Datacenter | Equal-cost paths к internet exchange |
| Enterprise (без NAT) | Routed public IP blocks через WAN |
| Lab/Testing | Простая агрегация bandwidth без сложности PCC |

---

## Преимущества

| Преимущество | Детали |
|--------------|--------|
| Простота | Минимальная конфигурация — достаточно добавить маршруты с одинаковым distance |
| Производительность | Без mangle rules, без connection marks — минимальная нагрузка на CPU |
| Масштабируемость | Легко добавлять WAN, добавляя маршруты |
| Быстрая convergence | check-gateway быстро удаляет мёртвые маршруты |
| Native routing | Использует стандартную таблицу маршрутизации — custom tables не нужны |

---

## Недостатки и риски

| Риск | Детали |
|------|--------|
| NAT breakage | Per-packet hashing нарушает симметрию connection tracking |
| Session instability | HTTPS, VPN, VoIP сессии могут сбрасываться mid-connection |
| No traffic policy | Невозможно направить конкретный трафик на конкретный WAN |
| Asymmetric routing | Обратный путь может отличаться, если upstream ISP имеют разную маршрутизацию |
| No bandwidth weighting | Все пути равны независимо от ёмкости |
| Firewall state issues | `connection-state=established` может не работать при смене пути |

---

## Типичные ошибки реализации

| Ошибка | Последствие | Исправление |
|--------|-------------|-------------|
| ECMP с masquerade NAT | Сломанные сессии, прерывистая связность | Использовать PCC |
| Отсутствие check-gateway | Мёртвые маршруты остаются active, blackhole трафик | Добавить `check-gateway=ping` ко всем маршрутам |
| Разные distance | Используется только один маршрут, load balancing не работает | Убедиться, что все маршруты имеют идентичный distance |
| ECMP + PCC одновременно | Конфликтующие routing decisions | Выбрать один метод на класс трафика |
| Нет firewall state rules | Invalid connection states | Добавить established/related accept rules первыми |

---

**Далее →** [PCC (Per Connection Classifier)](pcc.md)
