# SOC Alert Walkthrough 03: Snort ICMP Visibility Validation

**Lab:** Network Security Operations Lab
**Date:** 2026-05
**Status:** Authorized lab activity
**Classification:** True Positive

---

## Alert Summary

| Field | Detail |
|-------|--------|
| Alert source | Snort IDS |
| Alert type | ICMP traffic detected between lab segments |
| Trigger | ICMP ping sweep from Kali Linux to server VLAN |
| Target | Server VLAN (Windows Server 2016, Ubuntu NGINX) |
| Severity | Low to Medium |
| Decision | True Positive |

---

## Environment

| Component | Detail |
|-----------|--------|
| IDS | Snort receiving mirrored traffic via Cisco SPAN port |
| Switching | Cisco Catalyst with port mirroring configured |
| Firewall | Palo Alto PA-500 with inter-zone ICMP policy |
| Source | Kali Linux on attacker VLAN segment |
| Targets | Server VLAN hosts |

---

## Trigger

An ICMP ping sweep was executed from the Kali Linux endpoint to validate that Snort was correctly receiving and alerting on traffic traversing VLAN boundaries. This was a visibility validation test, not an attack.

```bash
nmap -sn [SERVER_SUBNET]
```

---

## Data Sources

| Source | What It Provided |
|--------|-----------------|
| Snort IDS | ICMP signature alerts with source and destination details |
| Palo Alto firewall logs | Inter-zone ICMP traffic entries |
| Wazuh SIEM | Correlated events from endpoint agents where ICMP reached the host |

---

## Observed Evidence

Snort generated ICMP-related alerts for each host that responded to the ping sweep. Alert entries showed source IP, destination IP, ICMP type and code, and the matching rule. Palo Alto inter-zone logs confirmed the traffic traversed the firewall between the attacker and server zones.

Hosts that did not respond were either offline or had ICMP blocked by the host-level firewall, which was noted as expected behavior and not a detection gap.

---

## Triage Steps

1. Reviewed Snort alert log for ICMP rule triggers.
2. Identified source IP and confirmed it matched the Kali endpoint on the attacker segment.
3. Noted which destination IPs responded and which did not.
4. Cross-referenced with Palo Alto logs to confirm inter-zone traversal was logged.
5. Verified that the SPAN port configuration was passing traffic correctly by comparing the list of responding hosts in Snort against the actual ping results.
6. Documented any hosts visible in ping results but missing from Snort alerts as a potential visibility gap.

---

## True Positive / False Positive Decision

**Decision: True Positive**

Snort correctly detected ICMP traffic between VLANs. The detection confirmed that the port mirroring configuration was working as intended and that Snort had visibility into inter-VLAN traffic. In a production environment, a ping sweep originating from an internal host would trigger investigation regardless of whether ICMP alone is considered malicious.

---

## Impact

| Area | Assessment |
|------|------------|
| IDS visibility confirmed | Yes, Snort detected inter-VLAN ICMP via SPAN |
| Firewall logging confirmed | Yes, Palo Alto logged inter-zone traffic |
| Coverage gaps | Any host with host firewall blocking ICMP will not appear in Snort alerts for this traffic type |

---

## Recommended Action (Production Context)

1. Document the alert as reconnaissance indicator
2. Investigate the source host for other recent suspicious activity
3. If unauthorized: isolate source, escalate to security team
4. Review firewall policy to confirm ICMP policy is intentional

---

## Lessons Learned

- SPAN port configuration must be validated through a live test before trusting IDS coverage. A misconfigured mirror port produces no alerts and no error, making the gap invisible.
- ICMP alone is low severity, but a ping sweep followed by a port scan is a common recon pattern. Both events should be documented together if they occur in sequence.
- Hosts that block ICMP are invisible to this detection method. IDS coverage for those hosts requires agent-based detection such as Wazuh.

---

## Screenshots / Redacted Evidence

| File | Content |
|------|---------|
| `evidence/03-snort-icmp-alerts.png` | Snort alert output for ICMP rules, IPs redacted |
| `evidence/03-palo-alto-icmp-log.png` | Firewall inter-zone log entries for ICMP, IPs redacted |
