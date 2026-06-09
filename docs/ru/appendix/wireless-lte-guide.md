# Приложение E — Руководство по Wireless WAN и LTE

> LTE/5G и wireless ISP как Multi-WAN линки.

---

## LTE как резервный WAN

```
/interface lte apn
set [find] apn=internet authentication=chap password=secret user=lteuser

/interface lte
set [find] disabled=no

/ip dhcp-client
add interface=lte1 add-default-route=no use-peer-dns=no

/ip route
add dst-address=0.0.0.0/0 gateway=lte1 distance=3 check-gateway=ping comment="LTE backup only"
```

**Никогда не добавляйте LTE в PCC classifier.**

---

## Мониторинг LTE

```
/tool netwatch
add host=8.8.8.8 interval=30s timeout=5s \
    down-script={
        :log warning "Primary WANs down — LTE should be active"
    }

/interface lte monitor lte1 once
```

---

## Пороги качества сигнала

| RSSI (dBm) | Качество | Действие |
|------------|---------|--------|
| -50 to -70 | Excellent | Подходит для production backup |
| -70 to -85 | Good | Приемлемо для backup |
| -85 to -100 | Fair | Мониторить внимательно |
| < -100 | Poor | Не полагаться на этот линк |

```
/interface lte monitor lte1 once
# Проверить: rssi, rsrp, sinr
```

---

## Wireless WISP как WAN

```
/interface wireless
set wlan1 mode=station ssid=WISP-AP frequency=5180 band=5ghz-a/n/ac

/ip dhcp-client
add interface=wlan1 add-default-route=no use-peer-dns=no
```

Используйте выделенное 5GHz радио для WISP backhaul; отдельно от локального WiFi AP.

---

## Контроль расхода трафика на LTE

```
/queue simple
add name=lte-cap target=lte1 max-limit=50M/50M comment="Cap LTE bandwidth"

/ip firewall address-list
add list=blocked-on-lte address=0.0.0.0/0

/ip firewall mangle
add chain=prerouting out-interface=lte1 connection-bytes=500000000-0 \
    action=mark-connection new-connection-mark=block-lte passthrough=yes

/ip firewall filter
add chain=forward connection-mark=block-lte action=drop comment="Block large transfers on LTE"
```

---

**[← Common Mistakes](common-mistakes.md)**
