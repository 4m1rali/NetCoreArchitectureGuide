# Chapter 24 — Disaster Recovery & Backup

> Business continuity planning for Multi-WAN edge routers.

---

## 24.1 Backup Strategy

| Type | Frequency | Storage | Recovery Time |
|------|-----------|---------|---------------|
| Config export (.rsc) | Daily | FTP + offsite | 5–15 minutes |
| Binary backup (.backup) | Weekly | USB + offsite | 2–5 minutes |
| Full Netinstall image | Monthly | Lab device | 30–60 minutes |
| Documentation (IP plan) | On change | Git/wiki | Reference |

---

## 24.2 Automated Backup Script

```
/system script
add name=full-backup source={
    :local date [/system clock get date]
    :local time [/system clock get time]
    :local name ("backup-" . $date . "-" . $time)
    /export file=$name
    /system backup save name=$name
    :log info ("Backup created: $name")
}

/system scheduler
add name=backup-daily interval=1d on-event=full-backup start-time=01:00:00
add name=backup-weekly interval=7d on-event=full-backup start-time=01:30:00
```

---

## 24.3 Recovery Procedures

### Scenario A — Config Corruption

```
/import file=latest-backup.rsc
# Review changes, reboot if needed
/system reboot
```

### Scenario B — Hardware Failure (Same Model Replacement)

```
1. Install RouterOS same version on new hardware
2. Apply license key for new software-id
3. /import file=latest-backup.rsc
4. Verify WAN links, routes, PCC
5. Update ARP on upstream switches if MAC changed
```

### Scenario C — Total Loss (Different Hardware)

```
1. Install RouterOS on available hardware
2. Manually reconfigure interfaces (names may differ)
3. Import backup — fix interface name mismatches
4. Test all WAN paths before production cutover
```

### Scenario D — Ransomware / Compromise

```
1. Netinstall (wipe device completely)
2. Install latest RouterOS
3. Import known-good backup from BEFORE compromise
4. Change ALL passwords
5. Review firewall rules for backdoors
6. Disable compromised VPN keys
```

---

## 24.4 Documentation to Maintain Off-Router

| Document | Content |
|----------|---------|
| IP Address Plan | All WAN/LAN IPs, gateways, subnets |
| ISP Contact List | Account numbers, NOC phone, circuit IDs |
| Password Vault | Admin, VPN, SNMP, API credentials |
| Network Diagram | Physical topology with port mapping |
| Config Change Log | Date, change, engineer, reason |
| License Keys | Software-ID → License key mapping |

---

## 24.5 Recovery Time Objectives

| Scenario | RTO Target | RPO Target |
|----------|-----------|-----------|
| Config error | 15 minutes | 0 (rollback) |
| Hardware failure | 1 hour | 24 hours (daily backup) |
| ISP outage (single) | 10 seconds (failover) | 0 |
| ISP outage (all) | 4+ hours (ISP repair) | N/A |
| Total site disaster | 4–8 hours | 24 hours |

---

## 24.6 Spare Hardware Strategy

| Tier | Spare | Storage |
|------|-------|---------|
| SOHO | Same model on shelf | Office |
| Enterprise | Pre-configured cold spare | Same building |
| ISP | Hot standby router (VRRP) | Same rack |
| Datacenter | Identical CCR pre-staged | Same DC |

---

**Next →** [Appendix C: FAQ](../appendix/faq.md)
