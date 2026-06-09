# Chapter 9 — Advanced NAT & DNS

> Hairpin NAT, inbound load balancing, DNS strategies, and port forwarding at scale.

---

## 9.1 Hairpin NAT (NAT Loopback)

Allows LAN clients to access internal servers using the **public WAN IP** from inside the network.

### Problem

```
LAN client → 203.0.113.2 (public IP of web server)
  → Router receives on LAN
  → No dst-nat rule matches (packet from LAN, not WAN)
  → Connection fails
```

### Solution

```
/ip firewall nat
add chain=dstnat in-interface-list=LAN dst-address=203.0.113.2 protocol=tcp \
    dst-port=80,443 action=dst-nat to-addresses=192.168.1.10

add chain=srcnat in-interface-list=LAN dst-address=192.168.1.10 \
    action=masquerade comment="Hairpin NAT"
```

### Per-WAN Hairpin

```
add chain=dstnat in-interface-list=LAN dst-address=203.0.113.2 action=dst-nat to-addresses=192.168.1.10
add chain=dstnat in-interface-list=LAN dst-address=198.51.100.2 action=dst-nat to-addresses=192.168.1.10
add chain=srcnat in-interface-list=LAN dst-address=192.168.1.10 action=masquerade
```

---

## 9.2 Inbound Load Balancing (DNAT Failover)

Inbound traffic cannot be PCC-balanced (ISP routing determines ingress). Use **DNS round-robin** or **BGP** for inbound distribution.

### DNS Round-Robin

```
# DNS zone on router or external DNS server
web.example.com  A  203.0.113.2   (ISP-1)
web.example.com  A  198.51.100.2  (ISP-2)
```

### Active-Passive Inbound Failover

Monitor primary WAN; update DNS TTL dynamically via script when failover occurs.

```
/tool netwatch
add host=203.0.113.1 interval=10s timeout=3s \
    down-script={
        /ip firewall nat disable [find comment="DNAT ISP-1"]
        /ip firewall nat enable [find comment="DNAT ISP-2 backup"]
    } \
    up-script={
        /ip firewall nat enable [find comment="DNAT ISP-1"]
        /ip firewall nat disable [find comment="DNAT ISP-2 backup"]
    }
```

---

## 9.3 DNS Strategies in Multi-WAN

| Strategy | Configuration | Pros | Cons |
|----------|--------------|------|------|
| Router DNS cache | `/ip dns set allow-remote-requests=yes` | Single resolver for LAN | Router becomes DNS SPOF |
| Per-WAN DNS | DHCP client use-peer-dns on each WAN | ISP-optimized resolution | Inconsistent answers |
| Public DNS | servers=8.8.8.8,1.1.1.1 | Consistent, fast | No ISP CDN optimization |
| Split DNS | Internal zone on router, external via WAN | Best for enterprise | Complex |

### Recommended Production DNS

```
/ip dns
set allow-remote-requests=yes cache-size=4096KiB \
    servers=8.8.8.8,1.1.1.1,max-cache-ttl=1w

/ip dhcp-server network
set [find] dns-server=192.168.1.1
```

### DNS over PCC Problem

DNS queries are short-lived UDP connections — they may hit different WANs, getting different CDN endpoints.

**Fix:** Force DNS through router cache only (clients use 192.168.1.1, never external DNS directly).

```
/ip firewall filter
add chain=forward in-interface-list=LAN protocol=udp dst-port=53 \
    dst-address=!192.168.1.1 action=drop comment="Force DNS via router"
```

---

## 9.4 Port Forwarding at Scale

### Template per Service

```
/ip firewall nat
add chain=dstnat in-interface=ether1 protocol=tcp dst-port=443 \
    action=dst-nat to-addresses=192.168.1.10 to-ports=443 comment="HTTPS ISP-1"
add chain=dstnat in-interface=ether2 protocol=tcp dst-port=443 \
    action=dst-nat to-addresses=192.168.1.10 to-ports=443 comment="HTTPS ISP-2"
```

### Port Range Forwarding

```
add chain=dstnat in-interface=ether1 protocol=tcp dst-port=50000-50100 \
    action=dst-nat to-addresses=192.168.1.20 to-ports=50000-50100 comment="RTP ISP-1"
```

### Address List Based DNAT

```
/ip firewall address-list
add list=public-services address=192.168.1.10
add list=public-services address=192.168.1.20

/ip firewall nat
add chain=dstnat in-interface-list=WAN protocol=tcp dst-port=80,443 \
    action=dst-nat to-addresses=192.168.1.10
```

---

## 9.5 NAT Conflict Prevention

| Rule | Implementation |
|------|---------------|
| One masquerade per out-interface | Separate srcnat rules per WAN |
| Never global masquerade | `out-interface-list=WAN` alone is insufficient — use per-interface |
| Align NAT with connection mark | Verify mark → route → interface → NAT chain |
| Disable unused NAT rules | Disabled rules still confuse audits |
| Log NAT misses | `action=log` on catch-all srcnat rule (debug only) |

---

## 9.6 Src-NAT with Fixed Public IPs

When ISP provides static public IP blocks:

```
/ip firewall nat
add chain=srcnat out-interface=ether1 action=src-nat \
    to-addresses=203.0.113.2 comment="Fixed NAT ISP-1"
add chain=srcnat out-interface=ether2 action=src-nat \
    to-addresses=198.51.100.2 comment="Fixed NAT ISP-2"
```

Fixed src-nat enables inbound services without dynamic port mapping conflicts.

---

**Next Chapter →** [Chapter 10: QoS & Traffic Prioritization](../10-qos-traffic/README.md)
