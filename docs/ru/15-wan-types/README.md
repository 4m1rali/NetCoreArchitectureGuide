# Глава 15 — WAN Types (PPPoE, LTE, DHCP)

> Шаблоны конфигурации для различных типов ISP handoff.

---

## 15.1 Static IP WAN (Reference)

```
/ip address
add address=203.0.113.2/30 interface=ether1

/ip route
add dst-address=203.0.113.1/32 gateway=ether1 scope=10
add dst-address=0.0.0.0/0 gateway=203.0.113.1 distance=1 check-gateway=ping
```

---

## 15.2 DHCP WAN

```
/ip dhcp-client
add interface=ether2 use-peer-dns=no use-peer-ntp=yes add-default-route=no \
    script={
        :if ($bound=1) do={
            /ip route remove [find comment="DHCP-ISP2-default"]
            /ip route add dst-address=0.0.0.0/0 gateway=$"gateway-address" \
                distance=1 check-gateway=ping comment="DHCP-ISP2-default"
        }
    }
```

**Критично:** `add-default-route=no` — маршруты управляются вручную для Multi-WAN.

---

## 15.3 PPPoE WAN

```
/interface pppoe-client
add name=pppoe-isp1 interface=ether1 user=customer@isp1.net password=secret \
    add-default-route=no use-peer-dns=no disabled=no

/ip firewall mangle
add chain=forward protocol=tcp tcp-flags=syn out-interface=pppoe-isp1 \
    action=change-mss new-mss=1440 passthrough=yes

/ip route
add dst-address=0.0.0.0/0 gateway=pppoe-isp1 routing-table=to-WAN1 \
    distance=1 check-gateway=ping comment="PCC ISP-1 PPPoE"
```

### PPPoE Multi-WAN Notes

| Параметр | Значение |
|----------|----------|
| MTU | 1492 (auto) |
| MSS | 1440 (clamp required) |
| Gateway | Use `pppoe-isp1` interface name, not IP |
| Reconnect | PPPoE auto-reconnects — routes reactivate via check-gateway |

---

## 15.4 LTE WAN

```
/interface lte
set lte1 band=""

/interface lte apn
set [find] apn=internet authentication=chap password=secret user=lteuser

/ip dhcp-client
add interface=lte1 add-default-route=no use-peer-dns=no

/ip route
add dst-address=0.0.0.0/0 gateway=lte1 routing-table=to-WAN3 distance=1 \
    check-gateway=ping comment="PCC ISP-3 LTE"
```

### LTE as Backup Only

```
/ip route
add dst-address=0.0.0.0/0 gateway=lte1 distance=3 check-gateway=ping comment="LTE backup"
```

Не включайте LTE в PCC classifier — используйте только как failover-only, чтобы избежать costly data usage.

---

## 15.5 VLAN WAN Handoff

```
/interface vlan
add name=vlan-isp1 vlan-id=100 interface=ether10
add name=vlan-isp2 vlan-id=200 interface=ether10

/ip address
add address=203.0.113.2/30 interface=vlan-isp1
add address=198.51.100.2/30 interface=vlan-isp2

/interface list member
add interface=vlan-isp1 list=WAN
add interface=vlan-isp2 list=WAN
```

Single physical port, multiple ISPs via VLAN tags.

---

## 15.6 Mixed WAN Type Design

| WAN | Type | Role | Method |
|-----|------|------|--------|
| ISP-1 | Fiber static | Primary | PCC bucket 0 + 1 (50%) |
| ISP-2 | PPPoE | Secondary | PCC bucket 2 + 3 (30%) |
| ISP-3 | LTE | Backup only | Failover distance=3, no PCC |

```
per-connection-classifier=both-addresses-and-ports:10/0
per-connection-classifier=both-addresses-and-ports:10/1
per-connection-classifier=both-addresses-and-ports:10/2
per-connection-classifier=both-addresses-and-ports:10/3
per-connection-classifier=both-addresses-and-ports:10/4
per-connection-classifier=both-addresses-and-ports:10/5
per-connection-classifier=both-addresses-and-ports:10/6
per-connection-classifier=both-addresses-and-ports:10/7
per-connection-classifier=both-addresses-and-ports:10/8
per-connection-classifier=both-addresses-and-ports:10/9
```

Buckets 0–4 → WAN1 (50%), 5–7 → WAN2 (30%), 8–9 → WAN2 overflow. LTE excluded from PCC.

---

**Следующая глава →** [Глава 16: Security Hardening](../16-security-hardening/README.md)
