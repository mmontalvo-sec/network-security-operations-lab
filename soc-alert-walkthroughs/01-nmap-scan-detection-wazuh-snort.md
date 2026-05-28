# SOC Alert Walkthrough 01: Nmap Scan Detection via Wazuh and Snort

**Lab:** Network Security Operations Lab
**Date:** 2026-05
**Status:** Authorized lab activity
**Classification:** True Positive

---

## Alert Summary

| Field | Detail |
|-------|--------|
| Alert source | Wazuh SIEM + Snort IDS |
| Alert type | Network reconnaissance detected |
| Trigger | Nmap TCP SYN scan from Kali Linux endpoint |
| Target | Windows Server 2016 and Ubuntu NGINX server |
| Severity | Medium |
| Decision | True Positive |

---

## Environment

| Component | Detail |
|-----------|--------|
| Firewall | Palo Alto PA-500 with segmented security zones |
| Switching | Cisco Catalyst with port mirroring configured to Snort sensor |
| SIEM | Wazuh, agents installed on Windows Server and Ubuntu targets |
| IDS | Snort, receiving mirrored traffic via SPAN port |
| Attacker | Kali Linux endpoint on isolated lab segment |
| Targets | Windows Server 2016 (Active Directory, DNS, DHCP), Ubuntu NGINX server |
| Network | VLAN-segmented lab: management, server, workstation, and attacker zones |

---

## Trigger

An authorized Nmap SYN scan was run from the Kali Linux endpoint against the server VLAN subnet to validate detection coverage and confirm that Wazuh and Snort were generating alerts as expected.

Command used:

```bash
nmap -sS -p 1-1024 [SERVER_SUBNET] --reason
```

This was a controlled detection test, not an attack. All activity was performed within the lab environment on equipment under my administration.

---

## Data Sources

| Source | What It Provided |
|--------|-----------------|
| Snort IDS | ICMP and TCP scan signatures triggered on mirrored traffic |
| Wazuh alerts | Connection attempt events from Wazuh agent on target systems |
| Palo Alto firewall logs | Inter-zone traffic entries showing scan source IP and destination ports |
| Windows Security Event Log | Port-scan-related connection attempts on Windows target |

---

## Observed Evidence

**Snort alerts:**
Multiple Snort rules fired including signatures consistent with TCP SYN port scan patterns. Alerts showed the source IP of the Kali endpoint and destination ports across the scanned range. Alert timestamps were consistent with the scan execution window.

**Wazuh dashboard:**
Alert volume spike visible immediately after scan execution. Wazuh correlated connection attempt events from the Windows agent. Alert rule IDs consistent with reconnaissance-related activity.

**Firewall logs:**
Palo Alto logged inter-zone traffic from the attacker VLAN to the server VLAN. Entries showed the scan source, destination IPs, and ports attempted.

**No ingress from the internet.** All traffic was internal to the lab, consistent with east-west lateral movement simulation.

---

## Triage Steps

1. Identified the source IP in Snort alert details and cross-referenced against known lab asset inventory.
2. Confirmed the source IP belonged to the authorized Kali Linux endpoint on the attacker segment.
3. Reviewed Wazuh alert timeline to confirm the spike aligned with the scan execution time.
4. Checked Palo Alto logs to confirm traffic did not originate from outside the lab environment.
5. Reviewed Windows Security Event Log on the target for related connection events.
6. Documented the alert chain from IDS trigger through SIEM correlation.

---

## True Positive / False Positive Decision

**Decision: True Positive**

The detected activity was a real network scan. The source was authorized and the activity was expected, but the detection infrastructure correctly identified and alerted on the behavior. In a production environment, this scan from an internal host would represent a significant concern requiring immediate investigation of the source endpoint.

---

## Impact

| Area | Assessment |
|------|------------|
| Detection coverage | Confirmed: Snort detected the scan via port-mirrored traffic |
| SIEM correlation | Confirmed: Wazuh surfaced related events from the endpoint agent |
| Firewall visibility | Confirmed: Palo Alto inter-zone logs captured scan traffic |
| Gaps identified | None identified for this scan type in this environment |

---

## Recommended Action (Production Context)

In a production NOC/SOC environment, this alert would trigger:

1. Immediate isolation investigation of the source endpoint
2. Ticket creation documenting the source IP, scan time, and target scope
3. Notification to the asset owner and security team
4. Review of recent authentication events on the source host
5. Determination of whether the scan was authorized (pen test, vulnerability scanner) or unauthorized

---

## Lessons Learned

- Port mirroring on the Cisco switch is required for Snort to see traffic between VLANs. Without the SPAN configuration, the scan would have been invisible to the IDS.
- Wazuh agent-side events complement IDS network-level visibility. The combination provides both network and host perspective on the same event.
- Cross-referencing multiple data sources (IDS, SIEM, firewall logs) before making a triage decision is more reliable than relying on a single alert source.
- Documenting the expected result before running the test makes it easier to distinguish gaps from expected behavior.

---

## Screenshots / Redacted Evidence

Redacted screenshots are available in the `evidence/` folder of this repository.

| File | Content |
|------|---------|
| `evidence/01-snort-alert-nmap.png` | Snort alert output with source IP and rule IDs visible, destination network redacted |
| `evidence/01-wazuh-alert-spike.png` | Wazuh dashboard showing alert volume spike during scan window |
| `evidence/01-palo-alto-interzone.png` | Firewall log entries showing inter-zone scan traffic, internal IPs redacted |

*All screenshots are redacted to remove internal IP addresses, hostnames, and any identifying configuration details.*
