# Recursive Routing

> Разрешение доступности шлюза — основа надёжного Multi-WAN.

---

## Инженерное определение

**Recursive Routing** — процесс, при котором маршрутизатор определяет доступность адреса шлюза через другой маршрут, а не предполагает, что шлюз напрямую подключён. В MikroTik для этого используются атрибуты **scope** и **target-scope** на маршрутах.

Без recursive routing `check-gateway` не может проверить состояние шлюза на WAN-каналах без прямого подключения (PPPoE, metro Ethernet, VLAN handoff).

---

## Scope и Target-Scope — объяснение

| Атрибут | Значение | Значение |
|---------|----------|----------|
| `scope` | 10 | Маршрут используется для разрешения доступности шлюза |
| `target-scope` | default (30) | Маршрут используется для фактической пересылки пакетов |
| `scope` | 30 | Directly connected — используется для forwarding |

### Поток разрешения

```
Default route: 0.0.0.0/0 via 203.0.113.1 (target-scope=30)
  → Router must find how to reach 203.0.113.1
  → Host route: 203.0.113.1/32 via ether1 (scope=10)
  → 203.0.113.1 reachable via ether1
  → Default route: ACTIVE
  → check-gateway pings 203.0.113.1 via ether1
```

---

## Шаблоны конфигурации

### Static IP WAN (Directly Connected)

```
/ip route
add dst-address=203.0.113.1/32 gateway=ether1 scope=10
add dst-address=0.0.0.0/0 gateway=203.0.113.1 distance=1 check-gateway=ping
```

### PPPoE WAN (Interface as Gateway)

```
/interface pppoe-client
add name=pppoe-isp1 interface=ether1 user=isp1@provider password=secret

/ip route
add dst-address=0.0.0.0/0 gateway=pppoe-isp1 distance=1 check-gateway=ping
```

### Multi-Hop Metro Ethernet

```
/ip route
add dst-address=198.51.100.1/32 gateway=10.0.0.1 scope=10
add dst-address=10.0.0.0/24 gateway=ether2
add dst-address=0.0.0.0/0 gateway=198.51.100.1 distance=1 check-gateway=ping
```

### Recursive с несколькими monitored targets

```
/ip route
add dst-address=203.0.113.1/32 gateway=ether1 scope=10
add dst-address=8.8.8.8/32 gateway=203.0.113.1 scope=10 target-scope=10
add dst-address=0.0.0.0/0 gateway=8.8.8.8 distance=1 check-gateway=ping
```

Это мониторит доступность интернета, а не только L2-доступность шлюза.

---

## Сценарии использования

| Сценарий | Зачем Recursive |
|----------|-----------------|
| PPPoE WAN | Gateway — виртуальный интерфейс, не IP |
| Metro Ethernet | Gateway за подсетью провайдера |
| VLAN WAN handoff | Gateway в другой VLAN-подсети |
| GRE tunnel WAN | Gateway внутри туннеля |
| ISP с /30 subnet | Gateway не в connected route |

---

## Преимущества

| Преимущество | Детали |
|--------------|--------|
| Надёжный check-gateway | Работает с любым типом WAN handoff |
| Мониторинг на уровне интернета | Можно мониторить 8.8.8.8 вместо только шлюза |
| Гибкая топология | Поддерживает сложные ISP handoff designs |
| Точность failover | Мёртвые маршруты удаляются, когда шлюз действительно недоступен |

---

## Недостатки и риски

| Риск | Детали |
|------|--------|
| Неверный scope | Маршрут никогда не становится active |
| Circular resolution | Gateway route указывает на себя |
| Дополнительные маршруты | Больше конфигурации, чем при directly-connected |
| Сложность отладки | Требуется route print detail для диагностики |

---

## Диагностические команды

```
/ip route print detail where dst-address=0.0.0.0/0
/ip route print detail where dst-address~"203.0.113"
/ip route check 203.0.113.1
```

---

**Далее →** [Connection Tracking Deep Dive](connection-tracking-deep.md)
