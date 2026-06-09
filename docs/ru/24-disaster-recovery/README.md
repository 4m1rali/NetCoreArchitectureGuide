# Глава 24 — Disaster Recovery и резервное копирование

> Планирование непрерывности бизнеса для edge-роутеров Multi-WAN.

---

## 24.1 Стратегия резервного копирования

| Тип | Частота | Хранение | Время восстановления |
|------|-----------|---------|---------------|
| Config export (.rsc) | Ежедневно | FTP + offsite | 5–15 минут |
| Binary backup (.backup) | Еженедельно | USB + offsite | 2–5 минут |
| Полный Netinstall image | Ежемесячно | Lab device | 30–60 минут |
| Документация (IP plan) | При изменении | Git/wiki | Справочно |

---

## 24.2 Автоматизированный скрипт backup

```
/system script
add name=full-backup source={
    :local date [/system clock get date]
    :local time [/system clock get time]
    :local name ("backup-" . $date . "-" . $time)
    /export file=$name
    /system backup save name=$name
    :log info ("Backup created: $name")
}

/system scheduler
add name=backup-daily interval=1d on-event=full-backup start-time=01:00:00
add name=backup-weekly interval=7d on-event=full-backup start-time=01:30:00
```

---

## 24.3 Процедуры восстановления

### Сценарий A — Повреждение конфигурации

```
/import file=latest-backup.rsc
# Проверить изменения, перезагрузить при необходимости
/system reboot
```

### Сценарий B — Отказ оборудования (замена той же модели)

```
1. Установить RouterOS той же версии на новое оборудование
2. Применить license key для нового software-id
3. /import file=latest-backup.rsc
4. Проверить WAN-линки, маршруты, PCC
5. Обновить ARP на upstream switches при смене MAC
```

### Сценарий C — Полная потеря (другое оборудование)

```
1. Установить RouterOS на доступное оборудование
2. Вручную перенастроить интерфейсы (имена могут отличаться)
3. Импортировать backup — исправить несовпадения имён интерфейсов
4. Протестировать все WAN-пути перед production cutover
```

### Сценарий D — Ransomware / компрометация

```
1. Netinstall (полная очистка устройства)
2. Установить последний RouterOS
3. Импортировать проверенный backup ДО компрометации
4. Сменить ВСЕ пароли
5. Проверить firewall rules на backdoors
6. Отключить скомпрометированные VPN keys
```

---

## 24.4 Документация вне роутера

| Документ | Содержание |
|----------|---------|
| IP Address Plan | Все WAN/LAN IP, gateways, subnets |
| ISP Contact List | Номера аккаунтов, NOC phone, circuit IDs |
| Password Vault | Admin, VPN, SNMP, API credentials |
| Network Diagram | Физическая топология с mapping портов |
| Config Change Log | Дата, изменение, инженер, причина |
| License Keys | Software-ID → License key mapping |

---

## 24.5 Целевые показатели восстановления

| Сценарий | RTO Target | RPO Target |
|----------|-----------|-----------|
| Ошибка конфигурации | 15 минут | 0 (rollback) |
| Отказ оборудования | 1 час | 24 часа (daily backup) |
| Отказ ISP (один) | 10 секунд (failover) | 0 |
| Отказ ISP (все) | 4+ часа (ремонт ISP) | N/A |
| Полная катастрофа площадки | 4–8 часов | 24 часа |

---

## 24.6 Стратегия резервного оборудования

| Уровень | Резерв | Хранение |
|------|-------|---------|
| SOHO | Та же модель на полке | Офис |
| Enterprise | Pre-configured cold spare | То же здание |
| ISP | Hot standby router (VRRP) | Та же стойка |
| Datacenter | Идентичный CCR pre-staged | Тот же DC |

---

**Далее →** [Приложение C: FAQ](../appendix/faq.md)
