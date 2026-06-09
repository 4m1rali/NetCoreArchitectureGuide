# PCC (Per Connection Classifier)

> Отраслевой стандарт Multi-WAN load balancing на MikroTik.

---

## Инженерное определение

**PCC (Per Connection Classifier)** — техника классификации трафика, которая использует детерминированный hash параметров соединения (source IP, destination IP, source port, destination port, protocol) для назначения каждого нового соединения конкретному WAN-пути. После классификации соединение сохраняет назначенный путь на всё время жизни через connection marks.

PCC работает на **Layer 3/4** через mangle firewall и routing-mark system MikroTik.

---

## Внутренний поток роутера (пошагово)

```
1. NEW packet arrives (no connection table entry)
2. Mangle PREROUTING chain:
   a. Check: connection-mark is empty (new connection)
   b. Calculate PCC hash:
      hash = (src-ip XOR dst-ip XOR src-port XOR dst-port) MOD N
      where N = number of WAN links
   c. Result 0 → assign connection-mark "WAN1-conn"
      Result 1 → assign connection-mark "WAN2-conn"
      Result 2 → assign connection-mark "WAN3-conn"
   d. Set routing-mark based on connection-mark
3. Connection tracking: STORE connection-mark
4. Routing: use routing table matching routing-mark
   → "to-WAN1" / "to-WAN2" / "to-WAN3"
5. NAT: masquerade on correct out-interface
6. SUBSEQUENT packets (ESTABLISHED):
   a. Connection table provides stored connection-mark
   b. Mangle rules SKIPPED (mark already set)
   c. Same routing table, same NAT, same WAN path
7. Connection closes → entry removed from table
```

---

## Поведение в MikroTik RouterOS

### Шаблон PCC Mangle Rules

```
/ip firewall mangle
add chain=prerouting dst-address-type=!local in-interface-list=LAN \
    per-connection-classifier=both-addresses-and-ports:3/0 \
    action=mark-connection new-connection-mark=WAN1-conn passthrough=yes

add chain=prerouting dst-address-type=!local in-interface-list=LAN \
    per-connection-classifier=both-addresses-and-ports:3/1 \
    action=mark-connection new-connection-mark=WAN2-conn passthrough=yes

add chain=prerouting dst-address-type=!local in-interface-list=LAN \
    per-connection-classifier=both-addresses-and-ports:3/2 \
    action=mark-connection new-connection-mark=WAN3-conn passthrough=yes
```

### Синтаксис Classifier

```
per-connection-classifier=both-addresses-and-ports:N/M
```

| Параметр | Значение |
|----------|----------|
| `both-addresses-and-ports` | Hash input: src-ip + dst-ip + src-port + dst-port |
| `N` | Общее количество WAN links |
| `M` | Bucket index (0 to N-1) |

### Назначение Routing Mark

```
add chain=prerouting connection-mark=WAN1-conn in-interface-list=LAN \
    action=mark-routing new-routing-mark=to-WAN1 passthrough=yes

add chain=prerouting connection-mark=WAN2-conn in-interface-list=LAN \
    action=mark-routing new-routing-mark=to-WAN2 passthrough=yes

add chain=prerouting connection-mark=WAN3-conn in-interface-list=LAN \
    action=mark-routing new-routing-mark=to-WAN3 passthrough=yes
```

### Таблицы маршрутизации

```
/ip route
add dst-address=0.0.0.0/0 gateway=203.0.113.1 routing-table=to-WAN1 check-gateway=ping
add dst-address=0.0.0.0/0 gateway=198.51.100.1 routing-table=to-WAN2 check-gateway=ping
add dst-address=0.0.0.0/0 gateway=192.0.2.1 routing-table=to-WAN3 check-gateway=ping
```

---

## Сценарии использования

| Среда | Почему PCC |
|-------|------------|
| ISP Edge Router | Распределение subscriber traffic по upstream |
| Enterprise Branch | Session-stable load balancing для 50–500 пользователей |
| SOHO с 2 ISP | Надёжный dual-WAN без session drops |
| VoIP + Data mixed | Каждый вызов остаётся на одном пути (без mid-call switching) |
| Gaming / Streaming | Стабильная latency на сессию |

---

## Преимущества

| Преимущество | Детали |
|--------------|--------|
| Session stability | Connection mark сохраняется на всё время жизни сессии |
| NAT compatible | Корректно работает с masquerade и src-nat |
| Predictable distribution | Hash algorithm даёт ~равное распределение со временем |
| Per-WAN policy | Каждый WAN имеет свою таблицу маршрутизации и NAT rules |
| Production proven | Наиболее распространённый Multi-WAN метод на MikroTik |
| Firewall compatible | Stateful rules корректно работают с connection marks |

---

## Недостатки и риски

| Риск | Детали |
|------|--------|
| CPU overhead | Mangle rules обрабатывают каждое новое соединение |
| Configuration complexity | Требуется координация mangle + routing tables + NAT |
| Uneven real-time distribution | Короткоживущие соединения могут кластеризоваться на одном WAN |
| No bandwidth weighting | 3/0, 3/1, 3/2 даёт равное разделение (не учитывает ёмкость) |
| Rule ordering critical | Неверный порядок mangle ломает классификацию |
| Connection table size | Большое количество соединений увеличивает потребление памяти |

---

## Типичные ошибки реализации

| Ошибка | Последствие | Исправление |
|--------|-------------|-------------|
| Отсутствие `passthrough=yes` | Классифицируется только первый пакет, остальные без mark | Всегда устанавливать passthrough=yes на connection marks |
| Mangle после routing | Routing decision принимается до классификации | Mangle должен быть в prerouting, до routing |
| Нет отдельных routing tables | Весь трафик использует main table | Создать to-WAN1, to-WAN2, to-WAN3 tables |
| NAT без out-interface filter | Используется неверный WAN IP для translation | Сопоставлять NAT rules с конкретным out-interface |
| Classifier N/M mismatch | Неравномерное или отсутствующее распределение | N = total WANs, M = 0 to N-1 |
| Забыли established bypass | Re-classification активных сессий | Добавить условие `connection-mark=no-mark` для classifier rules |
| Нет check-gateway на PCC routes | Трафик отправляется на мёртвый WAN | Включить check-gateway на всех PCC routing table routes |

---

**Далее →** [Failover](failover.md)
