# Глава 2 — Core Concepts

> Глубокие инженерные объяснения ECMP, PCC, Failover и Load Balancing.

---

## Обзор

Эта глава охватывает четыре фундаментальных механизма, используемых в production Multi-WAN развёртываниях на MikroTik. Каждая концепция включает инженерное определение, внутренний поток роутера, поведение MikroTik, сценарии использования, плюсы/минусы и типичные ошибки реализации.

| Концепция | Документ |
|-----------|----------|
| ECMP Routing | [ecmp.md](ecmp.md) |
| PCC (Per Connection Classifier) | [pcc.md](pcc.md) |
| Failover | [failover.md](failover.md) |
| Load Balancing | [load-balancing.md](load-balancing.md) |
| Policy Routing | [policy-routing.md](policy-routing.md) |
| Recursive Routing | [recursive-routing.md](recursive-routing.md) |
| Connection Tracking Deep Dive | [connection-tracking-deep.md](connection-tracking-deep.md) |

---

## Карта связей между концепциями

```
                    ┌──────────────────┐
                    │  LOAD BALANCING  │
                    │   (Goal/Result)  │
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
        ┌─────▼─────┐  ┌────▼────┐  ┌──────▼──────┐
        │    PCC    │  │  ECMP   │  │  FAILOVER   │
        │ (Primary) │  │ (Alt.)  │  │ (Resilience)│
        └───────────┘  └─────────┘  └─────────────┘
```

**Load Balancing** — это цель. **PCC** и **ECMP** — методы её достижения. **Failover** обеспечивает непрерывность при отказе пути.

---

## Когда что использовать

| Сценарий | Основной метод | Вспомогательный метод |
|----------|----------------|----------------------|
| ISP с 2–3 upstream, session-sensitive приложения | PCC | Failover |
| Bulk transfer с высокой пропускной способностью, без NAT | ECMP | Failover |
| Один активный WAN с резервом | Только Failover | — |
| Enterprise branch с SLA | PCC + Failover | Per-WAN NAT |
| Datacenter outbound diversity | ECMP или BGP | Failover |

---

**Начать чтение →** [ECMP Routing](ecmp.md)
