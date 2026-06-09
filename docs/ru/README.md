# Руководство по Multi-WAN архитектуре MikroTik для Enterprise

> **Senior ISP Network Architect · Эксперт MikroTik · Enterprise Network Designer**
>
> Production-ready документация для развёртывания Multi-WAN в средах ISP / Enterprise / Datacenter на MikroTik RouterOS.

---

## Выбор языка / Language Selection / انتخاب زبان

| Язык | Код | Путь к документации |
|------|-----|---------------------|
| **English** | `EN` | [docs/en/](../en/) |
| **فارسی (Persian)** | `FA` | [docs/fa/](../fa/) |
| **Русский (Russian)** | `RU` | [docs/ru/](README.md) |

---

## Краткое резюме

Это руководство представляет собой полное инженерное пособие по проектированию, развёртыванию и эксплуатации **Multi-WAN сетей на MikroTik RouterOS** в production-масштабе.

После изучения документации вы сможете:

- Спроектировать и развернуть реальную **Multi-WAN** топологию с 2–3 ISP
- Реализовать **Load Balancing**, **Failover**, **ECMP** и **PCC** на production-уровне
- Управлять **NAT**, **Routing** и **Firewall** как единой интегрированной системой
- Диагностировать нестабильность, ошибки маршрутизации и конфликты NAT в рабочих сетях
- Выбирать правильную комбинацию методов для сценариев ISP, Enterprise или Datacenter

---

## Структура руководства (по папкам)

```
NetCoreArchitectureGuide/
├── README.md                          ← Главный индекс (English + выбор языка)
└── docs/
    ├── en/                            ← English (полное содержание)
    ├── fa/                            ← فارسی
    └── ru/                            ← Русский (вы здесь)
        ├── README.md
        ├── 01-network-architecture/
        ├── 02-core-concepts/
        │   ├── ecmp.md
        │   ├── pcc.md
        │   ├── failover.md
        │   ├── load-balancing.md
        │   ├── policy-routing.md
        │   ├── recursive-routing.md
        │   └── connection-tracking-deep.md
        ├── 03-comparison-table/
        ├── 04-production-design/
        ├── 05-mikrotik-configuration/
        ├── 06-debugging-troubleshooting/
        ├── 07-engineering-analysis/
        ├── 08-final-summary/
        ├── 09-advanced-nat-dns/
        ├── 10-qos-traffic/
        ├── 11-vpn-multiwan/
        ├── 12-ipv6-multiwan/
        ├── 13-bgp-multihoming/
        ├── 14-monitoring-operations/
        ├── 15-wan-types/
        ├── 16-security-hardening/
        ├── 17-case-studies/
        ├── 18-routeros-licensing/
        ├── 19-hardware-selection/
        ├── 20-migration-upgrade/
        ├── 21-high-availability/
        ├── 22-hotspot-captive-portal/
        ├── 23-automation-scripting/
        ├── 24-disaster-recovery/
        ├── 25-security-labs-cve/
        └── appendix/
            ├── glossary.md
            ├── cheat-sheet.md
            ├── faq.md
            ├── common-mistakes.md
            └── wireless-lte-guide.md
```

---

## Оглавление глав

| # | Глава | Описание | Ссылка |
|---|-------|----------|--------|
| 1 | **Network Architecture** | Multi-WAN дизайн, роль NAT, таблицы маршрутизации, connection tracking, L2/L3/L4, поток пакетов | [Читать →](01-network-architecture/README.md) |
| 2 | **Core Concepts** | Инженерный разбор: ECMP, PCC, Failover, Load Balancing | [Читать →](02-core-concepts/README.md) |
| 3 | **Comparison Table** | Enterprise-матрица сравнения методов | [Читать →](03-comparison-table/README.md) |
| 4 | **Production Design** | Реальный сценарий с 3 ISP, топология, поток трафика, NAT journey | [Читать →](04-production-design/README.md) |
| 5 | **MikroTik Configuration** | Production-ready RouterOS: WAN, PCC, Failover, ECMP, NAT, Firewall | [Читать →](05-mikrotik-configuration/README.md) |
| 6 | **Debugging & Troubleshooting** | Анализ реальных сбоев и инструменты диагностики MikroTik | [Читать →](06-debugging-troubleshooting/README.md) |
| 7 | **Engineering Analysis** | Экспертный взгляд: когда использовать ECMP, PCC, Failover и комбинированные стратегии | [Читать →](07-engineering-analysis/README.md) |
| 8 | **Final Architecture Summary** | Рекомендации по лучшей Multi-WAN архитектуре | [Читать →](08-final-summary/README.md) |

## Часть IV — Advanced Topics

| # | Глава | Описание | Ссылка |
|---|-------|----------|--------|
| 9 | **Advanced NAT & DNS** | Hairpin NAT, inbound load balancing, DNS strategies, port forwarding | [Читать →](09-advanced-nat-dns/README.md) |
| 10 | **QoS & Traffic** | Bandwidth management и приоритизация трафика | [Читать →](10-qos-traffic/README.md) |
| 11 | **VPN over Multi-WAN** | IPsec, WireGuard, L2TP на нескольких WAN | [Читать →](11-vpn-multiwan/README.md) |
| 12 | **IPv6 Multi-WAN** | Dual-stack и IPv6-only Multi-WAN | [Читать →](12-ipv6-multiwan/README.md) |
| 13 | **BGP Multi-Homing** | ISP-grade multi-homed connectivity | [Читать →](13-bgp-multihoming/README.md) |
| 14 | **Monitoring & Operations** | SNMP, Syslog, Netwatch, NOC dashboards | [Читать →](14-monitoring-operations/README.md) |
| 15 | **WAN Types** | PPPoE, LTE, DHCP шаблоны | [Читать →](15-wan-types/README.md) |
| 16 | **Security Hardening** | Firewall, DDoS mitigation, access control | [Читать →](16-security-hardening/README.md) |
| 17 | **Case Studies** | Production scenarios и lessons learned | [Читать →](17-case-studies/README.md) |

## Часть V — Справочник и эксплуатация

| # | Глава | Описание | Ссылка |
|---|-------|----------|--------|
| 18 | **RouterOS Licensing** | Уровни лицензий, CHR, матрица функций, путь обновления | [Читать →](18-routeros-licensing/README.md) |
| 19 | **Hardware Selection** | Выбор CCR/RB, CPU/RAM, планирование интерфейсов | [Читать →](19-hardware-selection/README.md) |
| 20 | **Migration & Upgrade** | Безопасная миграция RouterOS 6 → 7 | [Читать →](20-migration-upgrade/README.md) |
| 21 | **High Availability** | VRRP, dual router, BGP-based HA | [Читать →](21-high-availability/README.md) |
| 22 | **Hotspot** | Captive portal + PCC для гостевого WiFi | [Читать →](22-hotspot-captive-portal/README.md) |
| 23 | **Automation** | Скрипты, schedulers, REST API, оповещения | [Читать →](23-automation-scripting/README.md) |
| 24 | **Disaster Recovery** | Стратегия backup, процедуры восстановления, RTO/RPO | [Читать →](24-disaster-recovery/README.md) |
| 25 | **Security Labs & CVEs** | Lab-упражнения, справочник CVE, упущенные угрозы | [Читать →](25-security-labs-cve/README.md) |

## Приложение

| # | Документ | Ссылка |
|---|----------|--------|
| A | Glossary | [appendix/glossary.md](appendix/glossary.md) |
| B | Cheat Sheet | [appendix/cheat-sheet.md](appendix/cheat-sheet.md) |
| C | FAQ | [appendix/faq.md](appendix/faq.md) |
| D | Top 30 Common Mistakes | [appendix/common-mistakes.md](appendix/common-mistakes.md) |
| E | Wireless & LTE Guide | [appendix/wireless-lte-guide.md](appendix/wireless-lte-guide.md) |

---

## Целевая аудитория

| Роль | Сценарий использования |
|------|------------------------|
| ISP Network Engineer | Multi-homed upstream, распределение трафика, SLA failover |
| Enterprise Network Admin | Dual/triple WAN для филиалов и HQ |
| Datacenter Operator | Разнообразие исходящих путей и резервирование шлюзов |
| MikroTik Consultant | Production-шаблоны для клиентов |

---

## Обзор сценария

```
                    ┌─────────────┐
                    │   ISP-1     │
                    └──────┬──────┘
                           │ WAN1 (ether1)
┌──────────┐         ┌─────┴──────┐         ┌─────────────┐
│  LAN     │─────────│  MikroTik  │─────────│   ISP-2     │
│ Clients  │  ether5 │   CCR/RB   │ WAN2    └─────────────┘
│ Servers  │         │   Router   │ ether2
└──────────┘         │            │─────────┌─────────────┐
                     │            │ WAN3    │   ISP-3     │
                     └────────────┘ ether3  └─────────────┘
```

**Рассматриваемые методы:**

| Метод | Уровень | Назначение |
|-------|---------|------------|
| PCC | L3/L4 (Mangle + Routing) | Per-connection распределение нагрузки |
| ECMP | L3 (Routing) | Equal-cost multipath маршрутизация |
| Failover | L3 (Gateway monitoring) | Автоматический fallback WAN |
| NAT | L3/L4 | Трансляция адресов для каждого WAN-пути |

---

## Быстрый старт

1. Прочитайте [Главу 1 — Network Architecture](01-network-architecture/README.md) для понимания потока пакетов и connection tracking.
2. Изучите [Главу 2 — Core Concepts](02-core-concepts/README.md) для внутренней логики ECMP, PCC, Failover и Load Balancing.
3. Ознакомьтесь с [Главой 3 — Comparison Table](03-comparison-table/README.md) для выбора метода.
4. Следуйте [Главе 4 — Production Design](04-production-design/README.md) для эталонной топологии.
5. Примените скрипты из [Главы 5 — MikroTik Configuration](05-mikrotik-configuration/README.md) на вашем роутере.
6. Держите открытой [Главу 6 — Debugging](06-debugging-troubleshooting/README.md) при вводе в эксплуатацию.
7. Проверьте решения с помощью [Главы 7 — Engineering Analysis](07-engineering-analysis/README.md).
8. Завершите работу [Главой 8 — Summary](08-final-summary/README.md).
9. Изучите advanced topics (главы 9–17) и [Cheat Sheet](appendix/cheat-sheet.md) для production edge cases.
10. Ознакомьтесь со справочными главами (18–24), [FAQ](appendix/faq.md) и [Common Mistakes](appendix/common-mistakes.md).

---

## Production-рекомендация (TL;DR)

| Компонент | Рекомендуемый метод |
|-----------|---------------------|
| Распределение нагрузки | **PCC** (Per Connection Classifier) |
| Резервирование шлюзов | **Recursive Routing + check-gateway** |
| Резервный путь | **Failover** с мониторингом шлюзов |
| Высокая пропускная способность LAN | **ECMP** только когда session stickiness не требуется |
| NAT | **Per-WAN masquerade** с connection marks |
| Безопасность | **Stateful firewall** + anti-loop правила |

> **Лучшая production-комбинация:** PCC (основной) + Failover (мониторинг шлюзов) + Per-WAN NAT + Stateful Firewall

---

## Совместимость с RouterOS

| Параметр | Требование |
|----------|------------|
| Версия RouterOS | 7.x рекомендуется (6.x совместима с оговорками) |
| Оборудование | RB4011, серия CCR или эквивалент для высокой пропускной способности |
| Лицензия | Level 4+ для расширенных функций маршрутизации |
| Интерфейсы | Минимум 3 WAN + 1 LAN |

---

## Соглашения документации

- Блоки конфигурации **готовы к copy-paste** — без inline-комментариев внутри скриптов
- IP-адреса используют документационные диапазоны RFC 5737 (`203.0.113.0/24`, `198.51.100.0/24`)
- Все диаграммы выполнены в ASCII для универсального отображения в Notion, GitHub и plain Markdown
- Русская, английская и персидская версии полностью зеркалируют структуру друг друга

---

## Лицензия и использование

Документация распространяется по лицензии **[MIT License](../../LICENSE)**. Русское пояснение: [LICENSE.ru.md](../../LICENSE.ru.md).

Лицензия RouterOS на устройстве — отдельная, приобретается у MikroTik ([Глава 18](18-routeros-licensing/README.md)).

Адаптируйте IP-адреса, имена интерфейсов и параметры ISP под вашу среду перед production-развёртыванием.

---

**[Начать чтение — Глава 1: Network Architecture →](01-network-architecture/README.md)** | **[FAQ →](appendix/faq.md)** | **[License →](../../LICENSE)**
