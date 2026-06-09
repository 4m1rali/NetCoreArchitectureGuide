# Chapter 2 — Core Concepts

> Deep engineering explanations for ECMP, PCC, Failover, and Load Balancing.

---

## Overview

This chapter covers the four fundamental mechanisms used in MikroTik Multi-WAN production deployments. Each concept includes engineering definition, internal router flow, MikroTik behavior, use cases, pros/cons, and common implementation errors.

| Concept | Document |
|---------|----------|
| ECMP Routing | [ecmp.md](ecmp.md) |
| PCC (Per Connection Classifier) | [pcc.md](pcc.md) |
| Failover | [failover.md](failover.md) |
| Load Balancing | [load-balancing.md](load-balancing.md) |
| Policy Routing | [policy-routing.md](policy-routing.md) |
| Recursive Routing | [recursive-routing.md](recursive-routing.md) |
| Connection Tracking Deep Dive | [connection-tracking-deep.md](connection-tracking-deep.md) |

---

## Concept Relationship Map

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

**Load Balancing** is the objective. **PCC** and **ECMP** are methods to achieve it. **Failover** ensures continuity when a path fails.

---

## When to Use What

| Scenario | Primary Method | Supporting Method |
|----------|---------------|-----------------|
| ISP with 2–3 upstreams, session-sensitive apps | PCC | Failover |
| High-throughput bulk transfer, no NAT | ECMP | Failover |
| Single active WAN with backup | Failover only | — |
| Enterprise branch with SLA requirements | PCC + Failover | Per-WAN NAT |
| Datacenter outbound diversity | ECMP or BGP | Failover |

---

**Start Reading →** [ECMP Routing](ecmp.md)
