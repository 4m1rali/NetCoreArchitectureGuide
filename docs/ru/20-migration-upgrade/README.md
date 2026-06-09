# Глава 20 — Миграция и обновление (RouterOS 6 → 7)

> Безопасный путь миграции существующих Multi-WAN развёртываний на RouterOS 7.

---

## 20.1 Зачем мигрировать на RouterOS 7

| Функция | ROS 6 | ROS 7 |
|---------|-------|-------|
| Таблицы маршрутизации | `/routing mark` | `/routing table` (чище) |
| BGP | Legacy instance model | Template + connection model |
| ECMP per-connection | Недоступен | `ecmp-per-connection=yes` |
| WireGuard | Ограниченный | Нативный, полная поддержка |
| VRF | Базовый | Полная поддержка VRF |
| IPv6 firewall | Общий с IPv4 | Отдельный `/ipv6 firewall` |
| Container | Нет | Docker-подобные containers |
| Производительность | Хорошая | На 20–40% лучше на том же железе |
| Долгосрочная поддержка | Maintenance mode | Активная разработка |

---

## 20.2 Чеклист перед миграцией

| # | Задача | Команда |
|---|------|---------|
| 1 | Полный export конфигурации | `/export file=pre-migration-backup` |
| 2 | Binary backup | `/system backup save name=pre-migration` |
| 3 | Задокументировать текущие маршруты | `/ip route print detail` |
| 4 | Задокументировать mangle rules | `/ip firewall mangle print` |
| 5 | Задокументировать NAT rules | `/ip firewall nat print` |
| 6 | Записать уровень лицензии | `/system license print` |
| 7 | Сначала тест в lab | Никогда не обновляйте production напрямую |
| 8 | Запланировать maintenance window | 30–60 минут |

---

## 20.3 Ключевые изменения конфигурации

### Таблицы маршрутизации (главное изменение)

```
# RouterOS 6
/ip route rule
add src-address=192.168.1.0/24 action=lookup routing-mark=to-WAN1

/routing mark
add name=to-WAN1

/ip route
add dst-address=0.0.0.0/0 gateway=GW1 routing-mark=to-WAN1

# RouterOS 7
/routing table
add name=to-WAN1 fib

/ip route
add dst-address=0.0.0.0/0 gateway=GW1 routing-table=to-WAN1

# Mangle остаётся без изменений
add chain=prerouting connection-mark=WAN1-conn action=mark-routing \
    new-routing-mark=to-WAN1 passthrough=yes
```

### BGP (полный редизайн)

```
# RouterOS 6
/routing bgp instance
/routing bgp peer
/routing bgp network

# RouterOS 7
/routing bgp template
/routing bgp connection
/routing bgp network
```

### Firewall

```
# RouterOS 7 — IPv6 firewall отдельный
/ipv6 firewall filter
/ipv6 firewall nat
/ipv6 firewall mangle
```

---

## 20.4 Процедура миграции

```
STEP 1: Lab test с экспортированной конфигурацией
STEP 2: Обновить RouterOS (пока не RouterBOARD firmware)
        /system package update check-for-updates
        /system package update download
        /system reboot
STEP 3: Проверить авто-миграцию конфигурации
        /ip route print detail
        /routing table print
STEP 4: Вручную исправить сломанные правила
STEP 5: Тест распределения PCC
        /ip firewall mangle print stats
STEP 6: Тест failover (отключить один WAN)
STEP 7: Тест NAT
        /ip firewall connection print
STEP 8: Обновить RouterBOARD firmware при стабильности
        /system routerboard upgrade
        /system reboot
STEP 9: Мониторинг 24 часа перед закрытием maintenance
```

---

## 20.5 План отката

```
# Если миграция не удалась:
1. Netinstall с RouterOS 6.x
2. Импорт pre-migration backup:
   /import file=pre-migration-backup.rsc
3. Или восстановление binary backup:
   /system backup load name=pre-migration
```

Храните pre-migration backup на ноутбуке и USB-накопителе.

---

## 20.6 Типичные проблемы миграции

| Проблема | Причина | Решение |
|-------|-------|-----|
| PCC перестал работать | routing-mark vs routing-table | Создать записи `/routing table` |
| BGP-сессии down | Новый формат BGP config | Перенастроить по template model |
| IPv6 firewall отсутствует | Разделён с IPv4 | Пересоздать в `/ipv6 firewall` |
| FastTrack ломает PCC | Изменение поведения | Отключить FastTrack |
| Wireless config потерян | Wireless package отдельный | Установить wireless package в ROS 7 |
| Скрипты падают | Изменения синтаксиса | Проверить и обновить скрипты |

---

## 20.7 Миграция без простоя (dual router)

Для критичных production-сетей:

```
1. Развернуть второй роутер с RouterOS 7 + новой конфигурацией
2. Тщательно протестировать на отдельных портах
3. Перенести LAN gateway IP на новый роутер
4. Старый роутер становится hot standby
5. Сохранить старый роутер 30 дней для отката
```

---

**Следующая глава →** [Глава 21: High Availability](../21-high-availability/README.md)
