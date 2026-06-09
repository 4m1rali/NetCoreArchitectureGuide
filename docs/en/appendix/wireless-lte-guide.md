# Appendix E — Wireless WAN & LTE Guide

> LTE/5G and wireless ISP as Multi-WAN links.

---

## LTE as Backup WAN

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

**Never add LTE to PCC classifier.**

---

## LTE Monitoring

```
/tool netwatch
add host=8.8.8.8 interval=30s timeout=5s \
    down-script={
        :log warning "Primary WANs down — LTE should be active"
    }

/interface lte monitor lte1 once
```

---

## Signal Quality Thresholds

| RSSI (dBm) | Quality | Action |
|------------|---------|--------|
| -50 to -70 | Excellent | Production backup OK |
| -70 to -85 | Good | Acceptable for backup |
| -85 to -100 | Fair | Monitor closely |
| < -100 | Poor | Do not rely on this link |

```
/interface lte monitor lte1 once
# Check: rssi, rsrp, sinr
```

---

## Wireless WISP as WAN

```
/interface wireless
set wlan1 mode=station ssid=WISP-AP frequency=5180 band=5ghz-a/n/ac

/ip dhcp-client
add interface=wlan1 add-default-route=no use-peer-dns=no
```

Use dedicated 5GHz radio for WISP backhaul; separate from local WiFi AP.

---

## Data Usage Control on LTE

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
