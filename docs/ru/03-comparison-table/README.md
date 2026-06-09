# Глава 3 — Enterprise Comparison Table

> Профессиональная матрица сравнения методов для принятия решений по Multi-WAN.

---

## Матрица сравнения Multi-WAN методов

| Атрибут | ECMP | PCC | Failover | ECMP + Failover | PCC + Failover |
|---------|------|-----|----------|-----------------|----------------|
| **Тип метода** | Routing | Mangle + Routing | Route Priority | Combined | Combined |
| **Уровень OSI** | L3 | L3/L4 | L3 | L3 | L3/L4 |
| **Модель распределения** | Packet / Flow | Flow (Per-Connection) | None (Active/Standby) | Flow | Flow |
| **Session Stability** | Poor (per-packet) / Good (per-conn) | Excellent | N/A (single path) | Good | Excellent |
| **NAT Compatibility** | Poor | Excellent | Good | Moderate | Excellent |
| **Performance Impact** | Minimal | Low–Moderate | Minimal | Low | Low–Moderate |
| **CPU Usage** | Very Low | Moderate | Very Low | Low | Moderate |
| **Scalability** | High (add routes) | High (add buckets) | High | High | High |
| **Complexity Level** | Low | High | Low | Medium | High |
| **Failure Risk** | Medium (NAT breaks) | Low | Very Low | Low | Very Low |
| **Best Use Case** | Routed public IPs, no NAT | NAT environments, ISP/Enterprise | Backup WAN, critical uptime | Datacenter, BGP | **Production Multi-WAN (Recommended)** |

---

## Детальный анализ атрибутов

### Модель распределения нагрузки

| Метод | Модель | Объяснение |
|-------|--------|------------|
| ECMP (default) | Per-Packet | Каждый пакет хешируется индивидуально — быстрее, но ломает NAT |
| ECMP (per-conn) | Per-Flow | Каждое соединение хешируется один раз — лучше NAT compatibility |
| PCC | Per-Flow | Connection mark сохраняется на время жизни сессии |
| Failover | Active/Standby | Весь трафик на primary до отказа |

### Рейтинг Session Stability

| Метод | Рейтинг | Детали |
|-------|---------|--------|
| ECMP per-packet | ★☆☆☆☆ | Сессии постоянно ломаются с NAT |
| ECMP per-connection | ★★★☆☆ | Стабильно в рамках connection, нет per-WAN NAT control |
| PCC | ★★★★★ | Полная session stickiness с per-WAN NAT |
| Failover | ★★★★☆ | Стабильно до failover event (сессии сбрасываются один раз) |

### Рейтинг NAT Compatibility

| Метод | Рейтинг | Детали |
|-------|---------|--------|
| ECMP per-packet | ★☆☆☆☆ | Masquerade ломает return path |
| ECMP per-connection | ★★★☆☆ | Работает с single masquerade rule |
| PCC | ★★★★★ | Per-interface masquerade с connection marks |
| Failover | ★★★★☆ | Single active NAT rule достаточно |

### Сравнение CPU Usage

| Метод | Cost нового соединения | Cost на пакет | 10K Connections |
|-------|------------------------|-----------------|-----------------|
| ECMP | None | Hash only | ~0% CPU |
| PCC | Mangle evaluation | Mark lookup | ~5–15% CPU |
| Failover | Ping only | None | ~1% CPU |
| PCC + Failover | Mangle + ping | Mark lookup | ~5–20% CPU |

### Уровень сложности

| Метод | Config Items | Routing Tables | Mangle Rules | NAT Rules |
|-------|-------------|----------------|--------------|-----------|
| ECMP | 3–5 | 1 (main) | 0 | 1 |
| PCC | 15–25 | 4 (main + 3 WAN) | 6–9 | 3 |
| Failover | 3–5 | 1 (main) | 0 | 1 |
| PCC + Failover | 20–30 | 4 | 6–9 | 3 |

### Оценка риска отказа

| Метод | Уровень риска | Основной риск |
|-------|---------------|---------------|
| ECMP + NAT | **HIGH** | Session breakage, asymmetric routing |
| PCC | **LOW** | Misconfigured mangle order |
| Failover only | **VERY LOW** | Single point of bandwidth |
| PCC + Failover | **VERY LOW** | PCC orphan on failed WAN (mitigated with Netwatch) |

---

## Decision Matrix

| Ваше требование | Рекомендуемый метод |
|-----------------|---------------------|
| NAT + Load Balancing + Failover | **PCC + Failover** |
| Public IP routing, no NAT | **ECMP + Failover** |
| Maximum uptime, single WAN sufficient | **Failover only** |
| Simple lab / testing | **ECMP** |
| ISP с 3 upstream | **PCC + Failover + Per-WAN NAT** |
| Enterprise 100+ users | **PCC + Failover** |
| VoIP + Data mixed traffic | **PCC + Policy Routing + Failover** |
| Datacenter BGP multi-homed | **BGP + ECMP** |

---

## Матрица поддержки функций (MikroTik RouterOS)

| Функция | ECMP | PCC | Failover |
|---------|------|-----|----------|
| RouterOS 6.x | Yes | Yes | Yes |
| RouterOS 7.x | Yes (enhanced) | Yes | Yes |
| IPv6 | Yes | Yes | Yes |
| VRF | Yes | Yes | Yes |
| FastTrack compatible | Yes | Partial (bypasses mangle) | Yes |
| Hardware offload | Yes | No (mangle) | Yes |
| VLAN interfaces | Yes | Yes | Yes |
| PPPoE WAN | Yes | Yes | Yes |
| Bridge WAN | Not recommended | Not recommended | Yes |

---

**Следующая глава →** [Глава 4: Production Design](../04-production-design/README.md)
