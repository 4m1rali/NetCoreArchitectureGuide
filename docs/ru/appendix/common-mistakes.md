# Приложение D — Топ-30 типичных ошибок

> Избегайте этих ошибок в production Multi-WAN развёртываниях.

---

| # | Ошибка | Последствие | Решение |
|---|---------|-------------|-----|
| 1 | ECMP + masquerade NAT | Сессии обрываются случайно | Используйте PCC |
| 2 | Нет check-gateway на маршрутах | Мёртвый WAN остаётся активным | Добавьте `check-gateway=ping` |
| 3 | FastTrack с PCC | Mangle обходится | Отключите FastTrack |
| 4 | Отсутствует passthrough=yes | Маркируется только первый пакет | Всегда ставьте passthrough=yes |
| 5 | PCC rules ниже policy rules | VoIP идёт через PCC | Policy rules первыми |
| 6 | Один глобальный masquerade | Конфликт NAT на return | Per-interface masquerade |
| 7 | LTE в PCC classifier | Огромный счёт за трафик | LTE только для failover |
| 8 | Нет MSS clamp на PPPoE | Крупные пакеты не проходят | MSS clamp 1440 |
| 9 | WAN-интерфейсы в bridge | Невозможна Multi-WAN маршрутизация | Router mode на WAN |
| 10 | Нет anti-loop rule | Петля маршрутизации при failover | Drop WAN-to-WAN forward |
| 11 | Нет backup перед изменением | Невосстановимая конфигурация | Export перед каждым изменением |
| 12 | Тестирование в production | Простой в рабочие часы | Сначала lab test |
| 13 | Одинаковый distance на failover routes | Непредсказуемый приоритет | distance=1,2,3 |
| 14 | Нет recursive route для PPPoE | Gateway недоступен | Interface как gateway |
| 15 | DNS напрямую на ISP | Нестабильное разрешение имён | DNS через роутер |
| 16 | add-default-route=yes на DHCP | DHCP перезаписывает ваши маршруты | add-default-route=no |
| 17 | Неверный classifier N/M | WAN отсутствует в распределении | N=число WAN, M=0 до N-1 |
| 18 | Нет Netwatch на PCC routes | PCC отправляет на мёртвый WAN | Netwatch disable script |
| 19 | Firewall дропает established | Все сессии обрываются | Accept established первым |
| 20 | Winbox открыт на WAN | Нарушение безопасности | Ограничить LAN/VPN |
| 21 | NTP не настроен | Логи бесполезны для отладки | Включите NTP |
| 22 | Connection tracking слишком мал | Дропы в пиковые часы | Увеличьте max-entries |
| 23 | VRRP без preempt | Backup остаётся master после восстановления | preempt=yes |
| 24 | BGP без route filter | Полная таблица убивает роутер | Filter только default |
| 25 | QoS до PCC mangle | Неверный порядок классификации | PCC первым, QoS вторым |
| 26 | VPN без policy route | Трафик туннеля балансируется PCC | Принудительный routing-mark |
| 27 | Нет мониторинга | Сбои обнаруживают пользователи | SNMP + Netwatch |
| 28 | Обновление без backup | Откат невозможен | Export + binary backup |
| 29 | Игнорирование разницы MTU | Частичные загадочные сбои | Per-interface MTU + MSS |
| 30 | Level 3 license для BGP | BGP недоступен | Обновление до Level 4+ |

---

## Классификация по серьёзности

| Серьёзность | Ошибки |
|----------|----------|
| **CRITICAL** (немедленный простой) | #1, #2, #4, #6, #9, #16, #19 |
| **HIGH** (деградация сервиса) | #3, #5, #7, #8, #14, #18, #22, #26 |
| **MEDIUM** (безопасность/стабильность) | #10, #15, #20, #24, #27, #29 |
| **LOW** (операционные) | #11, #12, #13, #17, #21, #23, #25, #28, #30 |

---

**[← FAQ](faq.md)** | **[Cheat Sheet →](cheat-sheet.md)**
