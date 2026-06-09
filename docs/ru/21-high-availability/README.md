# Глава 21 — High Availability (резервирование роутеров)

> Устранение роутера как единой точки отказа в Multi-WAN дизайнах.

---

## 21.1 Варианты HA-архитектуры

| Метод | Время failover | Сложность | Стоимость |
|--------|--------------|------------|------|
| VRRP (Virtual Router Redundancy) | 1–3 секунды | Средняя | 2 роутера |
| Dual router manual failover | Минуты | Низкая | 2 роутера |
| BGP с двумя edge-роутерами | Секунды | Высокая | 2 роутера + BGP |
| Cloud CHR failover | Секунды | Средняя | 2 VM |

---

## 21.2 VRRP на MikroTik

```
/interface vrrp
add interface=bridge-lan vrid=1 priority=150 preempt=yes \
    authentication=ah2 password=vrrp-secret

/ip address
add address=192.168.1.1/24 interface=bridge-lan comment="LAN gateway (VRRP virtual)"
```

### Дизайн dual router VRRP

```
Router-A (Master)                    Router-B (Backup)
├── Priority: 150                    ├── Priority: 100
├── Все WAN-линки активны            ├── Все WAN-линки активны
├── PCC + Failover настроены         ├── Идентичная PCC + Failover config
├── VRRP Master                      ├── VRRP Backup
└── Обрабатывает трафик              └── Берёт на себя при отказе Master
```

Оба роутера работают с идентичной PCC-конфигурацией. VRRP защищает только **LAN gateway IP**.

---

## 21.3 Синхронизация конфигурации

Используйте **configuration export/import** или **The Dude** для синхронизации конфигураций:

```
# На master — scheduled export в общее хранилище
/system scheduler
add name=config-sync interval=1h on-event={
    /export file=master-config
    /tool fetch address=192.168.1.200 src-path=master-config.rsc \
        dst-path=backup-config.rsc mode=ftp upload=yes
}
```

---

## 21.4 Особенности WAN HA

| Аспект | Детали |
|--------|--------|
| Оба роутера нуждаются во всех WAN-линках | Физический кабель к каждому роутеру или switch между ISP и роутерами |
| NAT state не синхронизируется | Активные соединения обрываются при VRRP failover (~1–3 с) |
| PCC state не синхронизируется | Новые соединения перераспределяются после failover |
| BGP | Оба роутера могут peer — для WAN HA используйте BGP вместо VRRP |
| Connection tracking | Не разделяется между роутерами — потеря сессий ожидаема |

---

## 21.5 ISP-grade HA с BGP

```
Router-A (AS 65050)          Router-B (AS 65050)
├── BGP к ISP-1              ├── BGP к ISP-1
├── BGP к ISP-2              ├── BGP к ISP-2
├── Advertise PI space       ├── Advertise PI space
└── Active                   └── Standby (higher prepend)

ISP видит два пути к вашему префиксу — автоматический failover без VRRP.
```

---

**Следующая глава →** [Глава 22: Hotspot и Captive Portal](../22-hotspot-captive-portal/README.md)
