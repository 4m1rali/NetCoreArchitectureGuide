# Load Balancing

> Концептуальная модель и практическая реализация распределения Multi-WAN трафика.

---

## Инженерное определение

**Load Balancing** в контексте Multi-WAN — практика распределения исходящего (и опционально входящего) сетевого трафика по нескольким WAN links для максимизации использования bandwidth, снижения congestion на отдельных путях и улучшения общей пропускной способности сети.

Load balancing — это **цель** — PCC и ECMP являются **методами** её достижения.

---

## Модели распределения

| Модель | Уровень | Гранулярность | Session Safe |
|--------|---------|---------------|--------------|
| Per-Packet | L3 | Отдельные пакеты | No |
| Per-Flow / Per-Connection | L3/L4 | Целое соединение/сессия | Yes |
| Per-Route | L3 | Routing table decision | Depends |
| Weighted | L3/L4 | Пропорционально ёмкости | Yes (с PCC) |

---

## Внутренний поток роутера — Decision Tree Load Balancing

```
Traffic arrives at edge router
    ↓
Is load balancing required?
    ├── NO → Single WAN (failover only)
    └── YES
         ↓
    Is NAT required?
         ├── YES → Use PCC (per-connection)
         └── NO
              ↓
         Is session stability required?
              ├── YES → Use PCC or ECMP per-connection
              └── NO → ECMP per-packet acceptable
                   ↓
              Are WAN capacities equal?
                   ├── YES → Equal PCC classifier (3/0, 3/1, 3/2)
                   └── NO → Weighted PCC or policy routing
```

---

## Практический Load Balancing с PCC

### Равномерное распределение (3 WAN, 33/33/33)

```
per-connection-classifier=both-addresses-and-ports:3/0  → WAN1
per-connection-classifier=both-addresses-and-ports:3/1  → WAN2
per-connection-classifier=both-addresses-and-ports:3/2  → WAN3
```

### Weighted Distribution (2 WAN, 70/30)

Для каналов с неравной ёмкостью используйте несколько classifier buckets:

```
# WAN1 (70%) — buckets 0, 1, 2, 3, 4, 5, 6
per-connection-classifier=both-addresses-and-ports:10/0
per-connection-classifier=both-addresses-and-ports:10/1
per-connection-classifier=both-addresses-and-ports:10/2
per-connection-classifier=both-addresses-and-ports:10/3
per-connection-classifier=both-addresses-and-ports:10/4
per-connection-classifier=both-addresses-and-ports:10/5
per-connection-classifier=both-addresses-and-ports:10/6

# WAN2 (30%) — buckets 7, 8, 9
per-connection-classifier=both-addresses-and-ports:10/7
per-connection-classifier=both-addresses-and-ports:10/8
per-connection-classifier=both-addresses-and-ports:10/9
```

### Policy-Based Load Balancing

Направление конкретных типов трафика на конкретные WAN:

```
# VoIP → WAN1 (low latency)
add chain=prerouting protocol=udp dst-port=5060,5061 \
    action=mark-connection new-connection-mark=WAN1-conn

# Backup traffic → WAN3 (cheaper link)
add chain=prerouting connection-bytes=10000000-0 \
    action=mark-connection new-connection-mark=WAN3-conn
```

---

## Поведение в MikroTik RouterOS

### Что MikroTik может балансировать

| Тип трафика | Метод | Примечания |
|-------------|-------|------------|
| Outbound TCP | PCC | Полная session stickiness |
| Outbound UDP | PCC | Работает, shorter timeout |
| Outbound ICMP | ECMP или PCC | Низкий объём, обычно незначимо |
| Inbound (DNAT) | Policy routing | Требует per-WAN dst-nat rules |
| VPN tunnels | Policy routing | Один tunnel на WAN, не balanced |

### Что MikroTik не может балансировать

| Тип трафика | Причина |
|-------------|---------|
| Single TCP connection | Невозможно разделить одно соединение по WAN |
| Inbound initiated connections | ISP routing определяет ingress path |
| Encrypted VPN inside tunnel | Outer tunnel — одно соединение |
| Multicast | Не маршрутизируется по unequal paths |

---

## Сценарии использования

| Среда | Стратегия |
|-------|-----------|
| ISP (500+ subscribers) | PCC 3-way + per-WAN NAT + Netwatch failover |
| Enterprise (100 users) | PCC 2-way equal + failover |
| SOHO (10 devices) | PCC 2-way или simple failover |
| Datacenter outbound | ECMP per-connection + BGP |
| Branch с VoIP | Policy routing: VoIP→WAN1, Data→PCC |

---

## Преимущества

| Преимущество | Детали |
|--------------|--------|
| Bandwidth aggregation | Суммарная пропускная способность всех WAN links |
| Congestion reduction | Ни один link не несёт весь трафик |
| Cost optimization | Дешёвые links для bulk, premium для latency-sensitive |
| Resilience + distribution | В сочетании с failover — полное решение |
| Scalable | Добавление WAN links через расширение classifier buckets |

---

## Недостатки и риски

| Риск | Детали |
|------|--------|
| Not true bonding | 3x100Mbps ≠ 300Mbps для single connection |
| Measurement difficulty | Per-WAN utilization сложно мониторить без инструментов |
| DNS issues | Разные WAN могут resolve в разные CDN nodes |
| Geo-IP inconsistency | Внешние сервисы видят разные public IPs |
| SSL/TLS session issues | Certificate pinning может конфликтовать с IP changes |
| Gaming NAT type | Strict NAT на load-balanced connections |

---

## Типичные ошибки реализации

| Ошибка | Последствие | Исправление |
|--------|-------------|-------------|
| Ожидание ускорения single-stream | Пользователь не видит улучшения для downloads | Объяснить: balancing — across connections, не within |
| Нет per-WAN DNS | DNS resolves через wrong WAN | Установить per-WAN DNS или использовать router DNS cache |
| Игнорирование upload capacity | Download balanced, upload congested | Мониторить оба направления per WAN |
| Balancing без monitoring | Dead WAN всё ещё получает new connections | Комбинировать с check-gateway + Netwatch |
| Слишком много mangle rules | CPU bottleneck на high-connection routers | Оптимизировать rule count, использовать hardware offload |
| Missing connection-mark persistence | Каждый пакет re-classified | Проверить passthrough=yes и connection table |

---

**Следующая глава →** [Глава 3: Comparison Table](../03-comparison-table/README.md)
