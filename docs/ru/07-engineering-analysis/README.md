# Глава 7 — Engineering Analysis (Expert View)

> Стратегический анализ для принятия production Multi-WAN решений.

---

## 7.1 Когда ECMP уместен — и когда он опасен

### Подходящие условия

| Условие | Причина |
|---------|---------|
| No NAT (routed public IP blocks) | Translation symmetry не требуется |
| BGP multi-homed setup | Standard protocol-level ECMP |
| Datacenter outbound с /24+ public blocks | Каждый server имеет свой public IP |
| RouterOS 7 с `ecmp-per-connection=yes` | Улучшенная обработка сессий |
| Internal traffic между routers | Нет зависимости от connection tracking |
| Testing и lab environments | Простота важнее stability |

### Опасные условия

| Условие | Риск |
|---------|------|
| Masquerade NAT на всех WAN | **CRITICAL** — sessions break randomly |
| VoIP / SIP traffic | Mid-call path switching вызывает drops |
| HTTPS с session cookies | TLS sessions reset при смене path |
| VPN tunnels (IPsec/OpenVPN) | Tunnel renegotiation при каждой смене path |
| Online gaming | NAT type становится Strict, latency spikes |
| Financial / trading applications | Microsecond path changes вызывают order failures |
| Любое session-sensitive application | User-visible disconnections |

### Экспертное заключение

> **ECMP с NAT — самая частая причина Multi-WAN failure в production MikroTik deployments.** Используйте PCC, если у вас нет routed public IPs.

---

## 7.2 Почему PCC — реальный Multi-WAN стандарт

### Техническое обоснование

| Фактор | Преимущество PCC |
|--------|------------------|
| Connection mark persistence | Вся сессия на одном WAN — без mid-stream switching |
| Per-WAN routing tables | Независимый path control per ISP |
| Per-WAN NAT | Корректный masquerade per egress interface |
| Firewall compatibility | Stateful rules работают с connection marks |
| Deterministic hashing | Reproducible, testable distribution |
| MikroTik native support | Built-in `per-connection-classifier` — без external tools |
| ISP industry adoption | Standard deployment pattern worldwide |

### Почему не другие методы

| Метод | Почему не стандарт |
|-------|-------------------|
| NTH (deprecated) | Заменён PCC в RouterOS 6.30+ |
| ECMP + NAT | Session breakage — unacceptable в production |
| Policy routing only | Нет automatic distribution — manual per-subnet |
| OSPF/BGP для SOHO | Overkill, требует ISP cooperation |
| Third-party bonding | Требует matching equipment на обоих концах |

### Экспертное заключение

> **PCC — de facto стандарт для MikroTik Multi-WAN, потому что это единственный built-in метод, который одновременно решает load distribution, NAT compatibility и session stability.**

---

## 7.3 Когда Failover обязателен

### Обязательные сценарии

| Сценарий | Почему Failover Required |
|----------|--------------------------|
| Любой production Multi-WAN | Без failover dead WAN = blackholed traffic |
| ISP с SLA requirements | Downtime должен быть < 30 seconds |
| VoIP / telephony | Calls must survive WAN failure |
| Credit card processing | Transaction continuity required |
| Remote branch offices | Нет on-site technician для manual switch |
| LTE backup links | Backup активируется только при primary failure |

### Failover Alone (без Load Balancing)

Допустимо когда:

- Secondary WAN значительно медленнее (LTE backup)
- Cost difference между ISP делает balancing uneconomical
- Только 2 WAN и primary имеет sufficient capacity
- Regulatory requirement для redundant path (не aggregated bandwidth)

### Экспертное заключение

> **Failover не опционален в production — это обязательный companion к любому load balancing method. Deploy check-gateway на каждом WAN route без исключений.**

---

## 7.4 Лучшая Production-комбинация

### Рекомендуемая архитектура

```
┌─────────────────────────────────────────────┐
│           PRODUCTION STACK                   │
│                                              │
│   ┌─────────────────────────────────────┐   │
│   │  PCC (Primary Load Distribution)    │   │
│   │  • per-connection-classifier 3/0,1,2│   │
│   │  • Separate routing tables          │   │
│   │  • Per-WAN masquerade NAT           │   │
│   └─────────────────────────────────────┘   │
│                      +                       │
│   ┌─────────────────────────────────────┐   │
│   │  Failover (Gateway Monitoring)      │   │
│   │  • check-gateway=ping on all routes │   │
│   │  • Netwatch scripts for PCC disable │   │
│   │  • Main table backup routes         │   │
│   └─────────────────────────────────────┘   │
│                      +                       │
│   ┌─────────────────────────────────────┐   │
│   │  Stateful Firewall                  │   │
│   │  • established/related first        │   │
│   │  • Anti-loop WAN-to-WAN drop        │   │
│   │  • WAN input drop                   │   │
│   └─────────────────────────────────────┘   │
│                      +                       │
│   ┌─────────────────────────────────────┐   │
│   │  Policy Routing (Optional)        │   │
│   │  • VoIP → lowest latency WAN      │   │
│   │  • Servers → specific WAN         │   │
│   │  • Management → dedicated WAN     │   │
│   └─────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

### Что НЕ следует комбинировать

| Комбинация | Проблема |
|------------|----------|
| ECMP + PCC на same traffic | Conflicting routing decisions |
| PCC + ECMP на same traffic | Double classification |
| Failover без check-gateway | Dead routes stay active |
| PCC без per-WAN NAT | NAT conflict на return path |
| FastTrack + PCC | FastTrack bypasses mangle marks |

---

## 7.5 Рекомендации по производительности и стабильности

### Hardware Sizing

| Users | Router Model | RAM | Expected PCC CPU |
|-------|-------------|-----|-----------------|
| 1–50 | RB750Gr3 | 256MB | < 5% |
| 50–200 | RB4011 | 1GB | 5–15% |
| 200–1000 | CCR2004 | 4GB | 10–25% |
| 1000+ | CCR2116/CCR2216 | 16GB | 15–30% |

### Optimization Guidelines

| Область | Рекомендация |
|---------|--------------|
| Mangle rules | Minimum necessary — 6 rules для 3-WAN PCC |
| Connection tracking | Set appropriate TCP timeouts |
| FastTrack | Disable если используется PCC mangle |
| DNS | Use router DNS cache — reduce WAN DNS queries |
| Monitoring | Netwatch every 10s, not every 1s |
| Logging | Disable firewall debug logging в production |
| NTP | Always enable — accurate logs для troubleshooting |
| Backup | Export config before every change |

### Connection Tracking Tuning

```
/ip firewall connection tracking
set enabled=yes tcp-established-timeout=1d tcp-time-wait-timeout=10s udp-timeout=30s
```

---

## 7.6 ISP vs Enterprise vs Datacenter

| Аспект | ISP | Enterprise | Datacenter |
|--------|-----|------------|------------|
| Primary method | PCC + BGP | PCC + Failover | BGP + ECMP |
| WAN count | 3–10+ | 2–3 | 4+ |
| NAT | Per-subscriber | Per-WAN masquerade | None (public IPs) |
| Failover | Netwatch + BGP | check-gateway | BGP failover |
| Monitoring | SNMP + Netwatch | Netwatch + Syslog | BGP + SNMP |
| Complexity | Very High | High | Medium (with BGP) |
| Session sensitivity | High (subscribers) | High (users) | Low (stateless) |

---

**Следующая глава →** [Глава 8: Final Architecture Summary](../08-final-summary/README.md)
