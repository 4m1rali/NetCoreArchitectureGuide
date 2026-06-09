# Lab Exercises — практика Multi-WAN

> 12 изолированных lab-упражнений. Используйте CHR или резервное оборудование. Никогда на production.

---

## Настройка lab-среды

```
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│  Virtual ISP1│   │  Virtual ISP2│   │  Virtual ISP3│
│  203.0.113.0 │   │ 198.51.100.0 │   │  192.0.2.0   │
└──────┬───────┘   └──────┬───────┘   └──────┬───────┘
       │ ether1           │ ether2           │ ether3
       └──────────┬───────┴──────────────────┘
                  │
           ┌──────▼──────┐
           │  CHR Router │
           │  RouterOS 7 │
           └──────┬──────┘
                  │ ether4
           ┌──────▼──────┐
           │  Test LAN   │
           │ 192.168.1.0 │
           └─────────────┘
```

### Базовая конфигурация (для всех lab)

```
/interface list
add name=WAN
add name=LAN
/interface list member
add interface=ether1 list=WAN
add interface=ether2 list=WAN
add interface=ether3 list=WAN
add interface=ether4 list=LAN

/ip address
add address=203.0.113.2/30 interface=ether1
add address=198.51.100.2/30 interface=ether2
add address=192.0.2.2/30 interface=ether3
add address=192.168.1.1/24 interface=ether4
```

---

## Lab 1 — Basic PCC 3-WAN

**Цель:** Настроить PCC и проверить распределение ~33/33/33.

**Задачи:**
1. Создать routing tables `to-WAN1`, `to-WAN2`, `to-WAN3`
2. Добавить PCC routes с check-gateway
3. Настроить PCC mangle rules (3/0, 3/1, 3/2)
4. Добавить per-WAN masquerade NAT
5. Генерировать трафик с 3 LAN-клиентов одновременно

**Проверка:**
```
/ip firewall mangle print stats
/ip firewall connection print count-only where connection-mark="WAN1-conn"
/ip firewall connection print count-only where connection-mark="WAN2-conn"
/ip firewall connection print count-only where connection-mark="WAN3-conn"
/interface monitor-traffic ether1,ether2,ether3 once
```

**Критерии успеха:** Каждый WAN несёт 25–40% новых соединений.

---

## Lab 2 — Failover Simulation

**Цель:** Проверить автоматический failover при отказе одного WAN.

**Задачи:**
1. Завершить конфигурацию Lab 1
2. Запустить непрерывный ping с LAN: `/ping 8.8.8.8 count=1000`
3. Отключить ether1: `/interface disable ether1`
4. Измерить downtime
5. Включить ether1 и наблюдать восстановление

**Проверка:**
```
/log print where message~"route"
/tool netwatch print
```

**Критерии успеха:** Failover < 15 секунд. Потеря ping < 20 пакетов.

---

## Lab 3 — NAT Symmetry Test

**Цель:** Доказать, что per-WAN NAT сохраняет return path.

**Задачи:**
1. Настроить PCC (Lab 1)
2. С LAN-клиента открыть 10 HTTPS-сессий на разные сайты
3. Проверить connection table на согласованность NAT translation

**Проверка:**
```
/ip firewall connection print where protocol=tcp and dst-port=443
```

**Критерии успеха:** Каждое соединение показывает согласованный `repl-src-address`, соответствующий правильному публичному IP WAN.

---

## Lab 4 — Weighted PCC (70/30)

**Цель:** Настроить неравномерное распределение нагрузки.

**Задачи:**
1. Использовать `:10/0` через `:10/6` для WAN1, `:10/7` через `:10/9` для WAN2
2. Отключить WAN3 для этого lab
3. Сгенерировать 100 новых TCP-соединений

**Критерии успеха:** WAN1 ≈ 65–75% соединений, WAN2 ≈ 25–35%.

---

## Lab 5 — Policy Routing (VoIP Priority)

**Цель:** Направить VoIP-подсеть через WAN1, остальное — через PCC.

**Задачи:**
1. Создать address range 192.168.1.240/28 как «VoIP subnet»
2. Добавить mangle rule: UDP 5060, 10000-20000 → WAN1-conn (выше PCC rules)
3. Генерировать SIP-трафик из .240/28 и HTTP из .10-.50
4. Проверить, что VoIP всегда использует WAN1

**Критерии успеха:** 100% UDP 5060 соединений помечены WAN1-conn.

---

## Lab 6 — Security Hardening Audit

**Цель:** Найти и закрыть exposed services.

**Задачи:**
1. Выполнить `/ip service print`
2. Выполнить `/user print`
3. Сканировать роутер со стороны «WAN» (Nmap из виртуальной ISP-сети):
   - `nmap -sS -p 21,22,23,80,443,8291,8728,8729 <router-wan-ip>`
4. Применить hardening из [Главы 16](../16-security-hardening/README.md)
5. Повторное сканирование — все порты filtered/closed с WAN

**Критерии успеха:** Ноль открытых management-портов с WAN.

---

## Lab 7 — CVE-2018-14847 Awareness (Defensive)

**Цель:** Понять уязвимость Winbox и проверить статус патча.

> **CVE-2018-14847** позволял читать произвольные файлы через порт Winbox. Исправлено в RouterOS 6.42.1+ и 6.43+.

**Задачи:**
1. Развернуть OLD RouterOS 6.40.x CHR instance (только изолированный lab)
2. Проверить версию: `/system resource print`
3. Убедиться, что чтение файлов через Winbox возможно на непропатченной версии (только для исследования)
4. Обновить до последнего long-term: `/system package update`
5. Повторный тест — уязвимость закрыта

**Критерии успеха:** Понимание, почему **никогда** не следует expose Winbox на WAN и **всегда** нужно патчить.

---

## Lab 8 — Netwatch Script Injection Test

**Цель:** Обнаружить небезопасные Netwatch scripts.

**Задачи:**
1. Просмотреть все записи Netwatch: `/tool netwatch print`
2. Проверить, принимают ли scripts внешний input
3. Заменить небезопасные scripts параметризованными безопасными версиями
4. Проверить, что failover работает после hardening scripts

**Небезопасный паттерн:**
```
down-script="/ip route disable [find gateway=203.0.113.1]"
```

**Более безопасный паттерн:**
```
down-script={/ip route disable [find comment="PCC ISP-1"]}
```

**Критерии успеха:** Нет scripts с hardcoded IP, которые можно манипулировать.

---

## Lab 9 — IPv6 Firewall Gap

**Цель:** Доказать, что IPv4-only firewall оставляет IPv6 exposed.

**Задачи:**
1. Настроить IPv6 на всех интерфейсах
2. Применить только IPv4 firewall (без `/ipv6 firewall`)
3. Попытаться получить доступ с WAN через IPv6
4. Добавить `/ipv6 firewall filter` rules, зеркалирующие IPv4
5. Повторный тест — доступ заблокирован

**Критерии успеха:** IPv6 WAN input dropped после добавления IPv6 firewall.

---

## Lab 10 — Connection Table Exhaustion

**Цель:** Понять resource limits.

**Задачи:**
1. Установить `max-entries=10000` (только lab)
2. Генерировать соединения через `/tool traffic-generator` или flood ping
3. Мониторинг: `/ip firewall connection print count-only`
4. Наблюдать поведение при лимите — новые соединения dropped?
5. Восстановить `max-entries=262144`

**Критерии успеха:** Задокументировать CPU и RAM при 80% заполнении connection table.

---

## Lab 11 — Backup File Secret Exposure

**Цель:** Убедиться, что `.rsc` export содержит пароли в plaintext.

**Задачи:**
1. `/export file=test-backup`
2. Открыть файл — найти `password=`, `secret=`, `pre-shared-key=`
3. Внедрить redaction паролей перед хранением backup вне роутера
4. Сравнить `/system backup save` (binary, немного лучше) с export

**Критерии успеха:** Политика команды: никогда не хранить `.rsc` export незашифрованными вне роутера.

---

## Lab 12 — Full Multi-WAN + Security Integration

**Цель:** Production-ready lab, объединяющий все навыки.

**Задачи:**
1. PCC 3-WAN + Failover + Per-WAN NAT
2. Policy routing для management subnet
3. Полный security hardening ([Глава 16](../16-security-hardening/README.md) + Глава 25)
4. Настроен SNMP monitoring
5. Автоматизированный ежедневный backup script
6. Документировать всю конфигурацию с IP plan
7. Симулировать отказ WAN1 при активном трафике
8. Запустить security scan с WAN — ноль открытых портов
9. Export финальной конфигурации как deliverable template

**Критерии успеха:** Пройти все verification commands из глав 5, 6 и 16.

---

## Rubric оценки lab

| Оценка | Критерии |
|--------|----------|
| **Expert** | Все 12 lab выполнены, задокументированы, security scan чистый |
| **Advanced** | Lab 1–8 выполнены, failover < 10s |
| **Intermediate** | Lab 1–5 выполнены, PCC сбалансирован |
| **Beginner** | Lab 1–2 выполнены |

---

**Далее →** [CVE & Exploit Reference](cve-exploit-reference.md)
