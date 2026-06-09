# Chapter 13 — BGP Multi-Homing

> ISP-grade multi-homed internet connectivity with BGP on MikroTik.

---

## 13.1 When BGP Replaces PCC

| Scenario | Method |
|----------|--------|
| SOHO / Branch (2–3 ISP) | PCC + Failover |
| Enterprise with public /24+ | BGP multi-homing |
| ISP with own ASN | BGP to multiple upstreams |
| Datacenter | BGP + ECMP |

BGP is appropriate when you own **provider-independent IP space** and **ASN**, and ISPs agree to BGP peering.

---

## 13.2 BGP Multi-WAN Architecture

```
                    ┌──────────────┐
                    │   ISP-1 AS   │
                    │   65001      │
                    └──────┬───────┘
                           │ eBGP
                    ┌──────▼───────┐
                    │  MikroTik    │
                    │  AS 65050    │
                    │  203.0.113.0/24
                    └──────┬───────┘
                           │ eBGP
                    ┌──────▼───────┐
                    │   ISP-2 AS   │
                    │   65002      │
                    └──────────────┘
```

---

## 13.3 BGP Configuration

```
/routing bgp template
add name=isp-peers as=65050 router-id=203.0.113.2

/routing bgp connection
add name=isp1 remote.address=203.0.113.1 .as=65001 local.role=ebgp \
    templates=isp-peers
add name=isp2 remote.address=198.51.100.1 .as=65002 local.role=ebgp \
    templates=isp-peers

/routing filter rule
add chain=bgp-in rule="if (bgp-ospf-nssa-is-not-supported-yet) " disabled=yes
```

### Advertise Own Prefix

```
/routing bgp connection
set isp1 output.network=bgp-networks
set isp2 output.network=bgp-networks

/routing bgp network
add network=203.0.113.0/24 synchronize=no
```

---

## 13.4 BGP Path Selection and Load Balancing

| Method | How |
|--------|-----|
| Primary/Backup | AS-PATH prepend on backup ISP |
| Load sharing | Accept default from both, ECMP |
| Partial routes | ISP sends only default route |
| Full table | ISP sends complete internet table (requires RAM) |

### Prepend for Backup

```
/routing filter rule
add chain=bgp-out rule="set bgp-path prepend 65050,65050,65050" \
    comment="Prepend ISP-2 — make ISP-1 preferred"
```

---

## 13.4 BGP vs PCC Comparison

| Attribute | BGP | PCC |
|-----------|-----|-----|
| IP ownership | Own PI space required | Any (NAT) |
| ISP cooperation | BGP session required | Just internet connection |
| Inbound control | Yes (path selection) | No (DNS only) |
| Complexity | Very high | Medium |
| Failover | Automatic (route withdrawal) | check-gateway |
| Cost | ASN + IP block fees | Standard ISP fees |

---

## 13.5 BGP Monitoring

```
/routing bgp session print
/routing bgp advertisement print
/routing route print where bgp=yes
/tool traceroute address=8.8.8.8
```

---

**Next Chapter →** [Chapter 14: Monitoring & Operations](../14-monitoring-operations/README.md)
