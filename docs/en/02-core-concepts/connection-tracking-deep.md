# Connection Tracking — Deep Dive

> Advanced conntrack behavior for Multi-WAN engineers.

---

## Connection Tracking Table Structure

| Field | Multi-WAN Relevance |
|-------|-------------------|
| `protocol` | TCP/UDP/ICMP — timeout differs per protocol |
| `src-address` / `dst-address` | Original endpoints before NAT |
| `reply-src-address` / `reply-dst-address` | After NAT translation |
| `connection-mark` | PCC WAN assignment — persists entire session |
| `routing-mark` | Active routing table for this connection |
| `connection-state` | new / established / related / invalid |
| `timeout` | Remaining session lifetime |
| `orig-packets` / `repl-packets` | Traffic volume per direction |
| `orig-rate` / `repl-rate` | Real-time throughput |
| `helper` | FTP/SIP/H323 ALG — can break Multi-WAN |
| `fasttrack` | If true, mangle was bypassed |

---

## Timeout Tuning for Multi-WAN

Default timeouts may be too aggressive for production ISP/Enterprise scale.

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

| Timeout | Too Low Effect | Too High Effect |
|---------|---------------|-----------------|
| tcp-established | Active sessions dropped | Memory bloat |
| udp | DNS/voip interruptions | Stale entries accumulate |
| generic | ICMP-based tools fail | Table exhaustion |

---

## Connection Table Capacity Planning

| Users | Expected Connections | RAM Needed | max-entries |
|-------|---------------------|------------|-------------|
| 50 | 5,000–10,000 | 256MB | 131072 |
| 200 | 20,000–50,000 | 1GB | 262144 |
| 500 | 50,000–150,000 | 4GB | 524288 |
| 1000+ | 150,000–500,000 | 8–16GB | 1048576 |

### Monitor Table Usage

```
/ip firewall connection print count-only
/ip firewall connection tracking print
/system resource print
```

---

## ALG Helpers and Multi-WAN

Application Layer Gateways (helpers) open related connections that must follow the parent connection's WAN mark.

| Helper | Protocol | Multi-WAN Risk |
|--------|----------|---------------|
| FTP | TCP 21 | Data channel may use wrong WAN |
| SIP | UDP 5060 | RTP streams may break |
| H323 | TCP 1720 | Dynamic ports conflict with NAT |
| PPTP | TCP 1723 | GRE protocol bypasses connection tracking |

### Disable Helpers When Not Needed

```
/ip firewall service-port
set ftp disabled=yes
set sip disabled=yes
set h323 disabled=yes
set pptp disabled=yes
```

For VoIP on Multi-WAN, use **SIP over TCP** or **static RTP port ranges** with policy routing.

---

## Untracked Traffic

Traffic marked as `untracked` bypasses connection tracking entirely.

```
/ip firewall mangle
add chain=prerouting action=mark-connection new-connection-mark=\
    no-mark passthrough=yes connection-state=established
```

**Never use untracked for PCC-balanced traffic** — marks will not persist.

---

## Connection Mark Persistence Verification

```
/ip firewall connection print where src-address~"192.168.1" \
    and connection-mark!=""
```

Every active LAN connection should show a non-empty `connection-mark` in PCC deployments.

---

**Next Chapter →** [Chapter 3: Comparison Table](../03-comparison-table/README.md)
