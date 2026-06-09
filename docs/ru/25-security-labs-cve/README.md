# Глава 25 — Security Labs, CVE и упущенные угрозы

> Практические lab-среды, справочник уязвимостей MikroTik и пробелы в безопасности, которые редко закрывают в production Multi-WAN.

---

## Назначение главы

| Раздел | Аудитория | Цель |
|--------|-----------|------|
| [Lab Exercises](lab-exercises.md) | Инженеры, изучающие Multi-WAN | Безопасная практика в изолированной lab-среде |
| [CVE & Exploit Reference](cve-exploit-reference.md) | Security-команды, NOC | Знать уязвимости, патчить, обнаруживать |
| [Overlooked Security](overlooked-security.md) | Архитекторы, аудиторы | Закрыть пробелы, которые упускают другие |

> **Этическое уведомление:** Lab-упражнения и информация о CVE в этой главе предназначены **только для defensive hardening и обучения**. Никогда не тестируйте exploit'ы против production-сетей или систем, которыми вы не владеете. Всегда работайте в изолированных lab-средах.

---

## Security posture в контексте Multi-WAN

Multi-WAN роутеры — **высокоценные цели**:

- Находятся на периметре интернета с несколькими публичными IP
- Выполняют NAT для целых организаций
- Часто работают на устаревших версиях RouterOS
- Сервисы управления (Winbox, API, SSH) нередко доступны извне
- Scripts и Netwatch создают attack surface автоматизации

```
                    ATTACK SURFACE MAP
    ┌─────────────────────────────────────────────┐
    │              MikroTik Edge Router              │
    │                                              │
    │  [Winbox:8291]  [SSH:22]  [API:8728]        │ ← Management
    │  [SNMP:161]     [WWW]     [FTP]              │ ← Services
    │  [BGP:179]      [IPsec]   [WireGuard]        │ ← Protocols
    │  [DNS:53]       [NTP]     [Bandwidth-test]   │ ← Auxiliary
    │  [Hotspot]      [Userman] [Container]        │ ← Applications
    │  [Scripts]      [Netwatch] [Scheduler]       │ ← Automation
    │                                              │
    │  WAN1 ──── WAN2 ──── WAN3                   │ ← Multi-WAN
    └─────────────────────────────────────────────┘
```

---

## Содержание главы

| Документ | Описание |
|----------|----------|
| [Lab Exercises](lab-exercises.md) | 12 практических lab: PCC, failover, отладка NAT, security hardening |
| [CVE & Exploit Reference](cve-exploit-reference.md) | Исторические CVE MikroTik, impact, затронутые версии, mitigation |
| [Overlooked Security](overlooked-security.md) | 25+ редко закрываемых пробелов безопасности в Multi-WAN |

---

## Минимальный security baseline (перед любым lab или production)

```
/system package update check-for-updates
/system package update download
/system reboot

/ip service
set telnet disabled=yes
set ftp disabled=yes
set www disabled=yes
set api disabled=yes
set winbox address=192.168.1.0/24
set ssh address=192.168.1.0/24

/user
set admin password="strong-unique-password-min-16-chars"

/ip firewall filter
add chain=input in-interface-list=WAN action=drop place-before=0
```

---

## Рекомендуемая lab-среда

| Компонент | Спецификация |
|-----------|--------------|
| Hypervisor | VMware, Hyper-V или Proxmox |
| Образ роутера | CHR (Cloud Hosted Router) — бесплатной лицензии 1 Mbps достаточно |
| Версия RouterOS | Последний stable 7.x + одна старая 6.x для сравнения CVE |
| Virtual WAN | 3 виртуальные сети, имитирующие ISP |
| Virtual LAN | 1 сеть с тестовыми клиентами |
| Изоляция | **Без подключения к production-интернету** — используйте внутренние DNS/NTP |
| Snapshots | Snapshot перед каждым lab для мгновенного rollback |

---

## Workflow оценки безопасности

```
1. Inventory версий RouterOS на всех edge-роутерах
2. Сверка с базой CVE (раздел 25.2)
3. Установка отсутствующих патчей (long-term upgrade branch)
4. Аудит упущенных пунктов (раздел 25.3)
5. Запуск lab-упражнений для проверки failover после hardening
6. Документирование находок в change log
7. Ежеквартальная повторная оценка
```

---

**Начать →** [Lab Exercises](lab-exercises.md) | [CVE Reference](cve-exploit-reference.md) | [Overlooked Security](overlooked-security.md)
