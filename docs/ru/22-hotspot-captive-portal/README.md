# Глава 22 — Hotspot и Captive Portal в Multi-WAN

> Аутентификация гостевого WiFi с PCC load balancing.

---

## 22.1 Архитектура Hotspot + Multi-WAN

```
Guest WiFi (VLAN 10) → Hotspot → PCC → WAN1/WAN2/WAN3
Corporate LAN (VLAN 1) → Policy Routing → WAN1 (primary)
```

Изолируйте гостевой трафик в отдельном VLAN с собственным PCC-путём.

---

## 22.2 Конфигурация Hotspot

```
/interface bridge
add name=bridge-hotspot

/interface vlan
add name=vlan-guest vlan-id=10 interface=bridge-lan

/interface bridge port
add bridge=bridge-hotspot interface=vlan-guest

/ip pool
add name=hotspot-pool ranges=10.10.10.2-10.10.10.254

/ip hotspot profile
add name=hsprof hotspot-address=10.10.10.1 dns-name=login.wifi.local

/ip hotspot
add name=hotspot1 interface=bridge-hotspot address-pool=hotspot-pool profile=hsprof

/ip hotspot user
add name=guest password=guest1 profile=default
```

---

## 22.3 Интеграция Hotspot с PCC

```
/ip firewall mangle
add chain=prerouting in-interface=bridge-hotspot connection-mark=no-mark \
    dst-address-type=!local per-connection-classifier=both-addresses-and-ports:2/0 \
    action=mark-connection new-connection-mark=WAN2-conn passthrough=yes
add chain=prerouting in-interface=bridge-hotspot connection-mark=no-mark \
    dst-address-type=!local per-connection-classifier=both-addresses-and-ports:2/1 \
    action=mark-connection new-connection-mark=WAN3-conn passthrough=yes
```

Гостевой трафик балансируется между WAN2 + WAN3; корпоративный остаётся на WAN1.

---

## 22.4 Rate limiting Hotspot

```
/ip hotspot user profile
set default rate-limit=5M/20M shared-users=3
add name=premium rate-limit=20M/50M
```

---

## 22.5 Walled Garden (доступ без аутентификации)

```
/ip hotspot walled-garden
add dst-host=*.google.com
add dst-host=*.apple.com
add dst-host=captive.apple.com
```

---

**Следующая глава →** [Глава 23: Автоматизация и скрипты](../23-automation-scripting/README.md)
