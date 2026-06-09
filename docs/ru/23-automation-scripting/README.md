# Глава 23 — Автоматизация и скрипты

> Автоматизированное управление Multi-WAN с помощью скриптов RouterOS и schedulers.

---

## 23.1 Динамическое обновление маршрута на DHCP WAN

```
/system script
add name=dhcp-wan-update source={
    :local gw $"gateway-address"
    :local iface $"interface"
    /ip route remove [find comment="dynamic-dhcp-wan"]
    /ip route add dst-address=0.0.0.0/0 gateway=$gw routing-table=to-WAN2 \
        distance=1 check-gateway=ping comment="dynamic-dhcp-wan"
    :log info ("DHCP WAN updated: gw=$gw iface=$iface")
}

/ip dhcp-client
add interface=ether2 script=dhcp-wan-update add-default-route=no use-peer-dns=no
```

---

## 23.2 Автоматическая проверка баланса PCC

```
/system script
add name=pcc-rebalance-check source={
    :local w1 [/ip firewall connection print count-only where connection-mark="WAN1-conn"]
    :local w2 [/ip firewall connection print count-only where connection-mark="WAN2-conn"]
    :local w3 [/ip firewall connection print count-only where connection-mark="WAN3-conn"]
    :local total ($w1 + $w2 + $w3)
    :if ($total < 10) do={ :return }
    :local pct1 (($w1 * 100) / $total)
    :local pct2 (($w2 * 100) / $total)
    :local pct3 (($w3 * 100) / $total)
    :if (($pct1 < 20) || ($pct1 > 50)) do={
        :log warning ("PCC imbalance: WAN1=$pct1% WAN2=$pct2% WAN3=$pct3%")
    }
}

/system scheduler
add name=pcc-check interval=5m on-event=pcc-rebalance-check
```

---

## 23.3 Оповещение о WAN failover через Telegram/Email

```
/system script
add name=wan-alert source={
    :global wanDownMessage "WAN FAILURE DETECTED"
    /tool e-mail send to="noc@company.com" subject="WAN ALERT" body=$wanDownMessage
    :log error $wanDownMessage
}

/tool netwatch
add host=203.0.113.1 interval=10s timeout=3s down-script="/system script run wan-alert"
```

---

## 23.4 Плановый backup конфигурации на FTP

```
/system script
add name=backup-ftp source={
    :local date [/system clock get date]
    /export file=("auto-backup-" . $date)
    /tool fetch address=192.168.1.200 src-path=("auto-backup-" . $date . ".rsc") \
        mode=ftp user=backup password=secret upload=yes
}

/system scheduler
add name=nightly-backup interval=1d on-event=backup-ftp start-time=02:00:00
```

---

## 23.5 Отключение мёртвого WAN из PCC classifier

```
/system script
add name=disable-wan1-pcc source={
    /ip firewall mangle disable [find comment="PCC WAN1"]
    :log warning "WAN1 PCC disabled — gateway down"
}

/system script
add name=enable-wan1-pcc source={
    /ip firewall mangle enable [find comment="PCC WAN1"]
    :log info "WAN1 PCC re-enabled — gateway restored"
}

/tool netwatch
add host=203.0.113.1 interval=10s timeout=3s \
    down-script=disable-wan1-pcc up-script=enable-wan1-pcc
```

---

## 23.6 Мониторинг через REST API (RouterOS 7)

```
/tool fetch url="http://192.168.1.1/rest/ip/route" http-method=get \
    http-header-field="Authorization: Basic base64userpass" output=user
```

Используйте REST API для интеграции с Grafana, Prometheus или кастомными NOC dashboards.

---

**Следующая глава →** [Глава 24: Disaster Recovery](../24-disaster-recovery/README.md)
