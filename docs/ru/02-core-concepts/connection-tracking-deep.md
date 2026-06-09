# Connection Tracking — Deep Dive

> Продвинутое поведение conntrack для инженеров Multi-WAN.

---

## Структура таблицы Connection Tracking

| Поле | Значение для Multi-WAN |
|------|------------------------|
| `protocol` | TCP/UDP/ICMP — timeout различается по протоколу |
| `src-address` / `dst-address` | Исходные endpoints до NAT |
| `reply-src-address` / `reply-dst-address` | После NAT translation |
| `connection-mark` | PCC WAN assignment — сохраняется на всю сессию |
| `routing-mark` | Активная таблица маршрутизации для этого соединения |
| `connection-state` | new / established / related / invalid |
| `timeout` | Оставшееся время жизни сессии |
| `orig-packets` / `repl-packets` | Объём трафика по направлениям |
| `orig-rate` / `repl-rate` | Пропускная способность в реальном времени |
| `helper` | FTP/SIP/H323 ALG — может нарушить Multi-WAN |
| `fasttrack` | Если true, mangle был обойдён |

---

## Настройка timeout для Multi-WAN

Стандартные timeout могут быть слишком агрессивными для production ISP/Enterprise масштаба.

```
/ip firewall connection tracking
set enabled=yes \
    tcp-established-timeout=1d \
    tcp-time-wait-timeout=10s \
    tcp-close-timeout=10s \
    tcp-syn-sent-timeout=5s \
    tcp-syn-received-timeout=5s \
    udp-timeout=30s \
    udp-stream-timeout=3m \
    icmp-timeout=10s \
    generic-timeout=10m \
    max-entries=1048576
```

| Timeout | Эффект при слишком низком значении | Эффект при слишком высоком значении |
|---------|-----------------------------------|-------------------------------------|
| tcp-established | Активные сессии сбрасываются | Раздувание памяти |
| udp | Прерывания DNS/voip | Накопление устаревших записей |
| generic | ICMP-based tools fail | Исчерпание таблицы |

---

## Планирование ёмкости таблицы соединений

| Пользователи | Ожидаемые соединения | RAM | max-entries |
|--------------|---------------------|-----|-------------|
| 50 | 5,000–10,000 | 256MB | 131072 |
| 200 | 20,000–50,000 | 1GB | 262144 |
| 500 | 50,000–150,000 | 4GB | 524288 |
| 1000+ | 150,000–500,000 | 8–16GB | 1048576 |

### Мониторинг использования таблицы

```
/ip firewall connection print count-only
/ip firewall connection tracking print
/system resource print
```

---

## ALG Helpers и Multi-WAN

Application Layer Gateways (helpers) открывают связанные соединения, которые должны следовать WAN mark родительского соединения.

| Helper | Протокол | Риск для Multi-WAN |
|--------|----------|-------------------|
| FTP | TCP 21 | Data channel может использовать неверный WAN |
| SIP | UDP 5060 | RTP streams могут обрываться |
| H323 | TCP 1720 | Dynamic ports конфликтуют с NAT |
| PPTP | TCP 1723 | GRE protocol обходит connection tracking |

### Отключение helpers, когда они не нужны

```
/ip firewall service-port
set ftp disabled=yes
set sip disabled=yes
set h323 disabled=yes
set pptp disabled=yes
```

Для VoIP в Multi-WAN используйте **SIP over TCP** или **static RTP port ranges** с policy routing.

---

## Untracked Traffic

Трафик с меткой `untracked` полностью обходит connection tracking.

```
/ip firewall mangle
add chain=prerouting action=mark-connection new-connection-mark=\
    no-mark passthrough=yes connection-state=established
```

**Никогда не используйте untracked для PCC-balanced traffic** — marks не будут сохраняться.

---

## Проверка сохранения Connection Mark

```
/ip firewall connection print where src-address~"192.168.1" \
    and connection-mark!=""
```

Каждое активное LAN-соединение должно показывать непустой `connection-mark` в PCC deployments.

---

**Следующая глава →** [Глава 3: Comparison Table](../03-comparison-table/README.md)
