# فصل ۲ — مفاهیم پایه

> توضیحات مهندسی عمیق برای ECMP، PCC، Failover و Load Balancing.

---

## نمای کلی

این فصل چهار مکانیزم بنیادی استفاده‌شده در استقرارهای Production Multi-WAN میکروتیک را پوشش می‌دهد. هر مفهوم شامل تعریف مهندسی، جریان داخلی روتر، رفتار MikroTik، موارد استفاده، مزایا/معایب و خطاهای رایج پیاده‌سازی است.

| مفهوم | سند |
|-------|-----|
| ECMP Routing | [ecmp.md](ecmp.md) |
| PCC (Per Connection Classifier) | [pcc.md](pcc.md) |
| Failover | [failover.md](failover.md) |
| Load Balancing | [load-balancing.md](load-balancing.md) |
| Policy Routing | [policy-routing.md](policy-routing.md) |
| Recursive Routing | [recursive-routing.md](recursive-routing.md) |
| Connection Tracking Deep Dive | [connection-tracking-deep.md](connection-tracking-deep.md) |

---

## نقشه ارتباط مفاهیم

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

**Load Balancing** هدف است. **PCC** و **ECMP** روش‌های دستیابی به آن هستند. **Failover** تداوم را هنگام قطع مسیر تضمین می‌کند.

---

## چه زمانی از چه چیزی استفاده کنیم

| سناریو | روش اصلی | روش پشتیبان |
|--------|----------|-------------|
| ISP با ۲–۳ Upstream، اپلیکیشن‌های حساس به Session | PCC | Failover |
| انتقال حجمی با Throughput بالا، بدون NAT | ECMP | Failover |
| یک WAN فعال با پشتیبان | فقط Failover | — |
| شعبه Enterprise با الزامات SLA | PCC + Failover | NAT به‌ازای هر WAN |
| تنوع خروجی Datacenter | ECMP یا BGP | Failover |

---

**شروع مطالعه ←** [ECMP Routing](ecmp.md)
