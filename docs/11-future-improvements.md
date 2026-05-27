# Future Improvements

This document tracks planned enhancements, next-phase work, and architectural improvements for the lab. Items are organized by priority and effort level.

---

## Phase 7 — Firewall Policy Hardening (Next Priority)

**Goal:** Replace broad any/any or permissive inter-zone rules with least-privilege policies.

**Planned work:**

- Document all required flows per zone pair (CLIENTS → WINDOWS-SERVERS, CLIENTS → SIEM, etc.)
- Create explicit allow rules for required traffic (DNS port 53, Kerberos 88, LDAP 389, RDP 3389, SMB 445, Wazuh agent 1514/1515, syslog 514)
- Create explicit deny-and-log rules for unauthorized inter-zone traffic
- Validate that allowed traffic passes and unauthorized traffic is blocked and logged
- Produce a policy matrix: source zone, destination zone, application, action, log setting

**Evidence to capture:**
- Palo Alto rulebase screenshot
- Security policy hit count after test period
- Traffic Monitor showing both allowed and denied events

---

## Phase 8 — Layer 2 Hardening

**Goal:** Harden the Cisco Catalyst 3560 at Layer 2 to reduce unauthorized access risk.

**Planned work:**

- Create a switch port map (port → device → VLAN → expected MAC)
- Configure port security on applicable access ports (max MAC count, violation action)
- Disable unused ports
- Document expected violation behavior and recovery procedure
- Validate a safe port security test (plug unauthorized device into secured port)

**Note:** Port security violation mode should be set to `restrict` or `protect` initially during lab testing to avoid hard lockouts.

---

## Phase 9 — Controlled Detection Scenarios

**Goal:** Generate and document authorized security events, correlate them across log sources, and produce IR-style findings documents.

**Planned scenarios:**

| Scenario | Systems | Expected Detections |
|---|---|---|
| Repeated failed domain logons | Client, DC | Event 4625 → Wazuh alert |
| SMB share enumeration | Client, DC | Events 5140/5145 → Wazuh |
| Internal nmap scan (authorized) | Kali/test host → lab | Snort alert + firewall log |
| Firewall policy deny test | Blocked traffic source | Palo Alto deny log → Wazuh |
| DNS query spike | Client | Wazuh DNS rule alert |
| Wazuh agent disconnect | Any endpoint | Wazuh manager alert |
| Snort rule trigger (safe payload) | Test host | Snort alert → Wazuh |
| Kerberos ticket anomaly | Client, DC | Event 4768/4769 → Wazuh |

**Deliverables per scenario:**
- Scenario definition (scope, systems, steps)
- Evidence screenshots
- Alert timeline from Wazuh
- Root cause analysis
- Analyst notes and recommendations

---

## Sysmon Deployment

Sysmon (System Monitor from Sysinternals) would significantly enhance endpoint visibility beyond the default Windows Security event log.

**Value:** Process creation (Event ID 1), network connections (Event ID 3), file creation (Event ID 11), DNS queries (Event ID 22).

**Plan:** Deploy Sysmon on domain clients and DC01 with a community ruleset (e.g., SwiftOnSecurity or Olaf Hartong config). Forward Sysmon events to Wazuh.

---

## Wazuh Custom Rules

The default Wazuh ruleset handles common Windows and IDS events but doesn't know the specific topology of this lab.

**Custom rules to develop:**

- Alert when a host in VLAN 10 (CLIENTS) attempts to reach VLAN 40 (SIEM) directly
- Alert on logon failures exceeding threshold within a time window
- Alert on new domain user creation outside of a maintenance window
- Alert when Snort alerts spike above baseline threshold

---

## Network Flow Baseline

Consider deploying ntopng, Zeek, or Netflow collection on the Snort sensor to capture flow-level data alongside Snort signature alerts.

Flow data helps answer questions that signatures alone cannot: "Is this volume of traffic from this host normal?" and "Is this the first time this source/destination pair has communicated?"

---

## Palo Alto Threat Prevention

If a Threat Prevention license is available or can be obtained for the PA-500, enabling App-ID and Content-ID would allow:

- Application-aware firewall rules (block evasion via non-standard ports)
- Anti-spyware and vulnerability protection profiles
- URL filtering (useful for simulating enterprise web policy)

---

## Log Retention Policy

Currently no formal log retention policy is defined. Future improvements:

- Define retention windows per log source (Wazuh, firewall, IDS)
- Configure Wazuh log rotation and archive settings
- Test archive log retrieval for IR simulation purposes

---

## Documentation Improvements

- Add numbered evidence index with screenshots for each completed phase
- Convert topology diagrams to rendered PNG alongside `.mmd` Mermaid source
- Add a MITRE ATT&CK mapping table for each planned detection scenario
- Create a project changelog documenting configuration changes over time
