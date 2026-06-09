# Policy Routing

> Выбор WAN-пути с учётом приложения и назначения.

---

## Инженерное определение

**Policy Routing** направляет определённый трафик на конкретные WAN-пути по критериям, отличным от одного лишь префикса назначения — исходная подсеть, протокол, порт, пользователь, время или объём байт соединения. В MikroTik policy routing реализуется через **mangle rules** (connection/routing marks), **routing rules** и **address lists**.

В отличие от PCC (который равномерно распределяет весь трафик), policy routing обеспечивает **детерминированный контроль пути**.

---

## Внутренний поток роутера

```
1. Пакет приходит из LAN
2. Mangle PREROUTING — policy rules оцениваются ПЕРВЫМИ (до PCC)
   a. Match: src-address=192.168.1.0/28 → mark WAN1-conn (VoIP subnet)
   b. Match: protocol=tcp dst-port=22 → mark WAN3-conn (management)
   c. Match: dst-address-list=streaming → mark WAN2-conn
3. Если policy match отсутствует → запускается PCC classifier (fallback distribution)
4. Routing mark применён → выбрана нужная таблица маршрутизации
5. NAT и forwarding выполняются как обычно
```

**Порядок правил критичен:** Policy rules должны располагаться **выше** PCC classifier rules.

---

## Шаблоны конфигурации MikroTik

### VoIP на WAN с минимальной задержкой

```
/ip firewall mangle
add chain=prerouting src-address=192.168.1.240/28 protocol=udp \
    dst-port=5060,5061,10000-20000 action=mark-connection \
    new-connection-mark=WAN1-conn passthrough=yes comment="VoIP → ISP-1"
```

### Управляющий трафик на выделенный WAN

```
/ip firewall mangle
add chain=prerouting src-address=192.168.1.0/28 protocol=tcp \
    dst-port=22,8291 action=mark-connection \
    new-connection-mark=WAN3-conn passthrough=yes comment="Mgmt → ISP-3"
```

### Принудительная маршрутизация конкретного пользователя на конкретный WAN

```
/ip firewall address-list
add list=user-john address=192.168.1.55

/ip firewall mangle
add chain=prerouting src-address-list=user-john action=mark-connection \
    new-connection-mark=WAN2-conn passthrough=yes
```

### Routing Rules (RouterOS 7)

```
/routing rule
add src-address=192.168.1.0/24 action=lookup-only-in-table table=to-WAN1
```

---

## Сценарии использования

| Среда | Политика |
|-------|----------|
| Enterprise | VoIP → fiber, bulk → cheaper WAN |
| ISP | Premium customers → low-latency upstream |
| Branch office | ERP system → primary WAN only |
| Gaming lounge | Gaming subnet → lowest latency ISP |
| Compliance | Financial traffic → audited WAN path |

---

## Преимущества

| Преимущество | Детали |
|--------------|--------|
| Гранулярный контроль | Per-subnet, per-protocol, per-user |
| Соответствие SLA | Критичные приложения всегда на лучшем пути |
| Оптимизация затрат | Bulk на дешёвом канале, realtime на premium |
| Совместимость с PCC | Policy first, PCC распределяет остаток |
| Аудит | Address lists документируют traffic policy |

---

## Недостатки и риски

| Риск | Детали |
|------|--------|
| Rule explosion | Слишком много правил → нагрузка на CPU |
| Maintenance burden | Изменение IP требует обновления списков |
| Bypass confusion | Пользователи в неправильной подсети получают неверный путь |
| Конфликты с PCC | Перекрывающиеся marks вызывают непредсказуемую маршрутизацию |
| Нет динамической адаптации | Статические правила не реагируют на деградацию WAN |

---

## Типичные ошибки

| Ошибка | Исправление |
|--------|-------------|
| Policy rules ниже PCC rules | Переместить policy rules в начало mangle |
| Одно соединение double-marked | Использовать `connection-mark=no-mark` только на PCC |
| Policy route без соответствующего NAT | Согласовать NAT out-interface с policy WAN |
| Забыли passthrough=yes | Mark применяется только к первому пакету |

---

**Далее →** [Recursive Routing](recursive-routing.md)
