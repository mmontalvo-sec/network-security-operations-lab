# Incident Response Methodology

## Overview

This document describes the incident response process used in the lab for analyzing
and documenting detection scenarios. The methodology follows a standard IR workflow:
identify → collect → analyze → timeline → root cause → remediation → lessons learned.

---

## IR Process Flow

```
1. IDENTIFY
   Alert fires in Wazuh, Snort, or firewall log
   Initial triage: is this real? noise? expected?

2. COLLECT EVIDENCE
   Screenshot the alert with timestamp
   Note source IP, destination, username, protocol
   Pull correlated logs from other sources

3. ANALYZE
   What happened? When? From where?
   What is the normal baseline for this behavior?
   Is this anomalous compared to baseline?

4. BUILD TIMELINE
   Sequence all related events chronologically
   Earliest indicator → chain of events → final state

5. ROOT CAUSE
   What caused the event?
   Was it authorized? Misconfiguration? Real threat?

6. REMEDIATION
   What would need to change to prevent recurrence?
   Policy update? Rule tune? Architecture change?

7. DOCUMENT
   Complete detection-test-template.md
   Store evidence in evidence/validation-results/
```

---

## Evidence Collection Standards

For each investigation, collect:

| Item | Source | Notes |
|---|---|---|
| Alert screenshot | Wazuh dashboard | Include timestamp, rule ID, agent |
| Raw event | Wazuh Events tab | Export or screenshot |
| Source IP | Alert fields | Identify the host |
| Destination | Alert fields | Identify the target |
| Username | Alert fields | Identify the account |
| Firewall log | PA Traffic Monitor | Did this cross a zone? |
| Snort alert | /var/log/snort/alert | Was there a scan? |
| Windows event | Event Viewer / Wazuh | What happened on the host? |

---

## Investigation Timeline Template

```
TIME (HH:MM:SS)   SOURCE         EVENT                          NOTES
----------------  -------------  -----------------------------  --------------------
HH:MM:SS          redteam1       nmap -sS 172.16.0.6           Scan initiated
HH:MM:SS          Snort          Port scan detected              Alert in snort/alert
HH:MM:SS          172.16.0.6    Event 4625 — Failed SMB logon  fakeuser, src .103
HH:MM:SS          172.16.0.6    Event 4625 — repeat            3 attempts total
HH:MM:SS          172.16.0.6    Event 4624 — Successful logon  aduser1, Type 3
HH:MM:SS          wserverbackup  Event 5140 — Share accessed    \\wserverbackup\Share
HH:MM:SS          wserverbackup  Event 5145 — Object checked    demo-alert.txt
HH:MM:SS          Wazuh          FIM alert — file created        demo-alert.txt hash
```

---

## Incident Report Fields

Every completed scenario should produce a report with:

- **Incident ID** — sequential number (IR-001, IR-002...)
- **Date and time window**
- **Systems involved**
- **Detection source** — which tool fired first?
- **Event summary** — what happened in plain language
- **Evidence collected** — list of screenshots and log snippets
- **Timeline** — chronological sequence
- **Root cause** — authorized test / misconfiguration / anomaly
- **Impact assessment** — what would this mean in a real environment?
- **Recommendations** — tuning, policy, architecture changes
- **Status** — closed / monitoring / remediated

Use `templates/incident-report-template.md` for each report.

---

## Triage Decision Matrix

| Condition | Action |
|---|---|
| Known authorized test, expected alert | Document as validated scenario — no further action |
| Unknown source, unexpected behavior | Treat as real — collect full evidence, escalate |
| Alert matches baseline | Add to noise documentation — consider tuning |
| Alert is new but plausible | Investigate fully before closing |
| Agent disconnected unexpectedly | Check host, check network path, check time sync |

---

## Lab-Specific IR Contacts

In this lab environment, all incidents are self-investigated. In a real SOC:

- Tier 1 analyst handles initial triage
- Tier 2 analyst handles full investigation
- IR lead handles escalation and reporting
- Management / legal notified based on impact

---

## Completed Scenario Reports

| Report | Scenario | Date | Status |
|---|---|---|---|
| IR-001 | Wazuh agent validation | Demo day | Closed — baseline confirmed |
| IR-002 | Nmap ping sweep | Demo day | Closed — authorized test |
| IR-003 | Nmap port scan | Demo day | Closed — authorized test |
| IR-004 | SMB failed authentication | Demo day | Closed — 4625 events confirmed |
| IR-005 | Successful login after failures | Demo day | Closed — correlation confirmed |
| IR-006 | SMB share access | Demo day | Closed — 5140/5145 confirmed |
| IR-007 | File integrity alert | Demo day | Closed — FIM triggered |
| IR-008 | Dashboard review / timeline | Demo day | Closed — 20,887 events documented |
