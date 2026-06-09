# Глава 18 — Лицензирование RouterOS

> Уровни лицензий MikroTik, требования к функциям и совместимость с Multi-WAN.

---

## 18.1 Обзор уровней лицензий

MikroTik RouterOS использует **программный уровень лицензии**, привязанный к каждому устройству. Лицензия определяет, какие расширенные функции доступны — это критично для проектирования Multi-WAN.

| Уровень | Название | Типичное применение | Пригодность для Multi-WAN |
|-------|------|-------------|----------------------|
| 3 | Level 3 | SOHO, базовая маршрутизация | Ограниченно — без BGP |
| 4 | Level 4 | Малый бизнес | **Минимум для production Multi-WAN** |
| 5 | Level 5 | Enterprise | Полный набор функций |
| 6 | Level 6 | ISP / Datacenter | Неограниченный BGP, полная таблица |

### Проверка текущей лицензии

```
/system license print
/system resource print
```

---

## 18.2 Доступность функций по уровню лицензии

| Функция | Level 3 | Level 4 | Level 5 | Level 6 |
|---------|---------|---------|---------|---------|
| Static routing | Yes | Yes | Yes | Yes |
| ECMP | Yes | Yes | Yes | Yes |
| PCC / Mangle | Yes | Yes | Yes | Yes |
| OSPF | No | Yes | Yes | Yes |
| BGP | No | Yes (limited) | Yes | Yes (unlimited) |
| MPLS | No | Yes | Yes | Yes |
| VRF | No | Yes | Yes | Yes |
| IPv6 full | Yes | Yes | Yes | Yes |
| WireGuard | Yes | Yes | Yes | Yes |
| IPsec | Yes | Yes | Yes | Yes |
| Hotspot | Yes | Yes | Yes | Yes |
| User Manager | No | Yes | Yes | Yes |
| The Dude | Yes | Yes | Yes | Yes |
| Container | No | No | Yes (ARM/x86) | Yes |

---

## 18.3 Минимальные требования для Multi-WAN

| Развёртывание | Мин. лицензия | Мин. оборудование | Причина |
|------------|-------------|--------------|--------|
| SOHO 2-WAN failover | Level 3 | hEX / RB750 | Достаточно базовой маршрутизации |
| Enterprise 3-WAN PCC | **Level 4** | RB4011 | VRF, OSPF при необходимости |
| ISP WISP 300 пользователей | **Level 5** | CCR2004+ | BGP, MPLS, высокое число соединений |
| Datacenter BGP | **Level 6** | CCR2116+ | Поддержка полной BGP-таблицы |
| Филиал VPN + PCC | Level 4 | RB4011 | IPsec + mangle |

---

## 18.4 Лицензия и BGP

| Уровень | Лимит BGP-маршрутов | BGP peers |
|-------|-----------------|-----------|
| 4 | ~1000 routes | 10 peers |
| 5 | ~4000 routes | 50 peers |
| 6 | Unlimited | Unlimited |

Для Multi-WAN только с BGP default route (без полной таблицы) **достаточно Level 4**.

Для полной интернет-таблицы BGP (750 000+ маршрутов) **обязательны Level 6 + 8GB+ RAM**.

---

## 18.5 Trial и лицензирование CHR

### Cloud Hosted Router (CHR)

| Лицензия CHR | Ограничение скорости | Multi-WAN |
|-------------|-------------|-----------|
| Free | 1 Mbps | Только lab |
| P1 ($45) | 1 Gbps | Небольшое развёртывание |
| P10 ($95) | 10 Gbps | Enterprise |
| P-Unlimited ($250) | Unlimited | ISP/Datacenter |

CHR идеален для виртуализированного lab-тестирования Multi-WAN и развёртываний на VMware/Hyper-V.

### Trial-лицензия

Новое оборудование MikroTik включает **60-дневный trial Level 6**. Используйте этот период для полного тестирования Multi-WAN до перехода на купленный уровень лицензии.

---

## 18.6 Путь обновления лицензии

```
Level 3 → Level 4: Купить ключ обновления у дистрибьютора MikroTik
Level 4 → Level 5: Тот же процесс
Level 5 → Level 6: Тот же процесс

/system license print
# Запишите software-id, купите ключ для этого ID
/system license renew account=your-mikrotik-account password=xxx license-key=KEY
```

---

## 18.7 Влияние лицензии на проектные решения

| Если лицензия... | Рекомендация по дизайну |
|-----------------|----------------------|
| Только Level 3 | Только failover — без BGP, без VRF |
| Level 4 | PCC + Failover — полноценный production |
| Level 5 | Добавить BGP, MPLS, продвинутый QoS |
| Level 6 | Полная ISP-архитектура с BGP multi-homing |

---

**Следующая глава →** [Глава 19: Выбор оборудования](../19-hardware-selection/README.md)
