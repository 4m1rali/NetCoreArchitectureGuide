# Chapter 25 — Security Labs, CVEs & Overlooked Threats

> Hands-on practice environments, MikroTik vulnerability reference, and security gaps rarely addressed in production Multi-WAN deployments.

---

## Purpose of This Chapter

| Section | Audience | Goal |
|---------|----------|------|
| [Lab Exercises](lab-exercises.md) | Engineers learning Multi-WAN | Safe practice in isolated lab |
| [CVE & Exploit Reference](cve-exploit-reference.md) | Security teams, NOC | Know vulnerabilities, patch, detect |
| [Overlooked Security](overlooked-security.md) | Architects, auditors | Close gaps others miss |

> **Ethics Notice:** Lab exercises and CVE information in this chapter are for **defensive hardening and education only**. Never test exploits against production networks or systems you do not own. Always operate in isolated lab environments.

---

## Security Posture in Multi-WAN Context

Multi-WAN routers are **high-value targets**:

- They sit at the internet edge with multiple public IPs
- They carry NAT for entire organizations
- They often have outdated RouterOS versions
- Management services (Winbox, API, SSH) are frequently exposed
- Scripts and Netwatch create automation attack surface

```
                    ATTACK SURFACE MAP
    ┌─────────────────────────────────────────────┐
    │              MikroTik Edge Router              │
    │                                              │
    │  [Winbox:8291]  [SSH:22]  [API:8728]        │ ← Management
    │  [SNMP:161]     [WWW]     [FTP]              │ ← Services
    │  [BGP:179]      [IPsec]   [WireGuard]        │ ← Protocols
    │  [DNS:53]       [NTP]     [Bandwidth-test]   │ ← Auxiliary
    │  [Hotspot]      [Userman] [Container]        │ ← Applications
    │  [Scripts]      [Netwatch] [Scheduler]       │ ← Automation
    │                                              │
    │  WAN1 ──── WAN2 ──── WAN3                   │ ← Multi-WAN
    └─────────────────────────────────────────────┘
```

---

## Chapter Contents

| Document | Description |
|----------|-------------|
| [Lab Exercises](lab-exercises.md) | 12 hands-on labs: PCC, failover, NAT debug, security hardening |
| [CVE & Exploit Reference](cve-exploit-reference.md) | Historical MikroTik CVEs, impact, affected versions, mitigation |
| [Overlooked Security](overlooked-security.md) | 25+ rarely-addressed security gaps in Multi-WAN |

---

## Minimum Security Baseline (Before Any Lab or Production)

```
/system package update check-for-updates
/system package update download
/system reboot

/ip service
set telnet disabled=yes
set ftp disabled=yes
set www disabled=yes
set api disabled=yes
set winbox address=192.168.1.0/24
set ssh address=192.168.1.0/24

/user
set admin password="strong-unique-password-min-16-chars"

/ip firewall filter
add chain=input in-interface-list=WAN action=drop place-before=0
```

---

## Recommended Lab Environment

| Component | Specification |
|-----------|--------------|
| Hypervisor | VMware, Hyper-V, or Proxmox |
| Router image | CHR (Cloud Hosted Router) — free 1Mbps license sufficient |
| RouterOS version | Latest stable 7.x + one old 6.x for CVE comparison |
| Virtual WANs | 3 virtual networks simulating ISPs |
| Virtual LAN | 1 network with test clients |
| Isolation | **No connection to production internet** — use internal DNS/NTP |
| Snapshots | Snapshot before each lab for instant rollback |

---

## Security Assessment Workflow

```
1. Inventory RouterOS version on all edge routers
2. Cross-reference with CVE database (Section 25.2)
3. Apply missing patches (long-term upgrade branch)
4. Audit overlooked items (Section 25.3)
5. Run lab exercises to verify failover still works post-hardening
6. Document findings in change log
7. Schedule quarterly re-assessment
```

---

**Start →** [Lab Exercises](lab-exercises.md) | [CVE Reference](cve-exploit-reference.md) | [Overlooked Security](overlooked-security.md)
