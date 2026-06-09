# Chapter 11 — VPN over Multi-WAN

> IPsec, WireGuard, and L2TP deployment across multiple WAN paths.

---

## 11.1 VPN and Multi-WAN — Core Rule

**A VPN tunnel is a single connection — it cannot be PCC-balanced across WANs.**

Each VPN tunnel must be bound to **one specific WAN interface**. Use policy routing to select the WAN, and deploy redundant tunnels on different WANs for resilience.

---

## 11.2 Architecture Pattern

```
                    ┌──────────────┐
                    │  VPN Server  │
                    │  (HQ/DMZ)    │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
        IPsec WAN1   WireGuard WAN2   L2TP WAN3
        (Primary)    (Backup)         (Mobile)
              │            │            │
              └────────────┼────────────┘
                           │
                    ┌──────▼───────┐
                    │   MikroTik   │
                    │  Branch CCR  │
                    └──────────────┘
```

---

## 11.3 IPsec over Specific WAN

```
/ip ipsec peer
add name=peer-hq address=203.0.113.100 local-address=203.0.113.2 \
    exchange-mode=ike2

/ip ipsec identity
add peer=peer-hq auth-method=pre-shared-key secret=strong-preshared-key

/ip ipsec policy
add src-address=192.168.1.0/24 dst-address=10.0.0.0/24 peer=peer-hq \
    tunnel=yes action=encrypt

/ip firewall mangle
add chain=prerouting dst-address=203.0.113.100 action=mark-routing \
    new-routing-mark=to-WAN1 passthrough=yes comment="IPsec via ISP-1"
```

---

## 11.4 WireGuard over Specific WAN

```
/interface wireguard
add name=wg-hq listen-port=51820 mtu=1420

/interface wireguard peers
add interface=wg-hq public-key="HQ_PUBLIC_KEY" endpoint-address=198.51.100.100 \
    endpoint-port=51820 allowed-address=10.0.0.0/24

/ip address
add address=10.0.0.2/24 interface=wg-hq

/ip firewall mangle
add chain=prerouting out-interface=wg-hq action=mark-routing \
    new-routing-mark=to-WAN2 passthrough=yes comment="WireGuard via ISP-2"
```

---

## 11.5 VPN Failover Between WANs

Deploy two tunnels to the same HQ endpoint via different WANs:

```
# Primary: IPsec via WAN1
# Backup: IPsec via WAN2 (disabled until WAN1 fails)

/tool netwatch
add host=203.0.113.1 interval=10s timeout=3s \
    down-script={
        /interface wireguard enable wg-hq-backup
        /ip route enable [find comment="VPN-backup-route"]
    } \
    up-script={
        /interface wireguard disable wg-hq-backup
        /ip route disable [find comment="VPN-backup-route"]
    }
```

---

## 11.6 Remote Access (Road Warrior) over Multi-WAN

Publish VPN on all WAN public IPs:

```
# L2TP/IPsec on all WAN interfaces
/interface l2tp-server server
set enabled=yes max-mtu=1450 max-mru=1450

/ip firewall nat
add chain=dstnat in-interface=ether1 protocol=udp dst-port=500,4500,1701 action=accept
add chain=dstnat in-interface=ether2 protocol=udp dst-port=500,4500,1701 action=accept
```

Clients connect to whichever public IP is reachable.

---

## 11.7 VPN Performance Tuning

| Setting | Value | Reason |
|---------|-------|--------|
| MTU | 1400–1420 | Avoid fragmentation over PPPoE/LTE WAN |
| MSS clamp | 1360 | VPN header overhead |
| keepalive | 10s | Fast dead tunnel detection |
| FastTrack | Disabled on VPN traffic | Preserve mangle marks |

```
/ip firewall mangle
add chain=forward out-interface=wg-hq protocol=tcp tcp-flags=syn \
    action=change-mss new-mss=1360 passthrough=yes
```

---

## 11.8 Common VPN + Multi-WAN Errors

| Error | Consequence | Fix |
|-------|-------------|-----|
| VPN without policy route | Tunnel traffic PCC-balanced → breaks | Force routing-mark to specific WAN |
| NAT on IPsec | Double NAT breaks IKE | Accept IPsec before masquerade |
| Same tunnel on two WANs | IKE rekey conflicts | One tunnel per WAN |
| No MSS clamp | Intermittent large packet failure | Clamp MSS on VPN interface |

---

**Next Chapter →** [Chapter 12: IPv6 Multi-WAN](../12-ipv6-multiwan/README.md)
