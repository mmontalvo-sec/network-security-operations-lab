# SOC Alert Walkthrough 02: Windows Failed Logon Event Review

**Lab:** Network Security Operations Lab
**Date:** 2026-05
**Status:** Authorized lab activity
**Classification:** True Positive (simulated brute-force pattern)

---

## Alert Summary

| Field | Detail |
|-------|--------|
| Alert source | Wazuh SIEM (Windows Security Event Log ingestion) |
| Alert type | Multiple failed authentication attempts on domain controller |
| Trigger | Repeated Event ID 4625 entries on Windows Server 2016 |
| Target | Windows Server 2016 running Active Directory |
| Severity | Medium to High (depending on account targeted) |
| Decision | True Positive |

---

## Environment

| Component | Detail |
|-----------|--------|
| SIEM | Wazuh, agent installed on Windows Server 2016 |
| Target | Windows Server 2016 — Domain Controller (AD, DNS, DHCP) |
| Log source | Windows Security Event Log via Wazuh agent |
| Attacker simulation | Repeated failed logon attempts from a domain-joined workstation |
| Network | VLAN-segmented lab environment |

---

## Trigger

Repeated failed logon attempts were generated against the domain controller from a domain-joined lab workstation using incorrect credentials. This simulated a basic brute-force or credential-stuffing scenario against a domain account.

This was a controlled test performed in the authorized lab environment.

---

## Data Sources

| Source | What It Provided |
|--------|-----------------|
| Wazuh SIEM | Correlated Event ID 4625 alerts from the Windows agent |
| Windows Security Event Log | Individual failed logon records with source workstation, account name, and failure reason |
| Active Directory (ADUC) | Account lockout status confirmation |

---

## Observed Evidence

**Windows Security Event Log entries:**

Repeated Event ID 4625 entries observed within a short time window. Each entry included:

| Field | Detail |
|-------|--------|
| Event ID | 4625 (An account failed to log on) |
| Account targeted | Domain user account (redacted) |
| Failure reason | Unknown username or bad password |
| Source workstation | Lab workstation (hostname and IP redacted) |
| Logon type | 3 (Network logon) |

**Wazuh alerts:**

Wazuh rule fired on repeated 4625 events exceeding the threshold for failed authentication attempts within a defined time window. The alert was categorized as an authentication failure pattern consistent with brute-force behavior.

**Account lockout:**

After the configured lockout threshold was exceeded, the targeted account was locked out. This was confirmed in Active Directory Users and Computers.

---

## Triage Steps

1. Identified the alert in the Wazuh dashboard and reviewed the raw event data.
2. Noted the targeted account name, source workstation, logon type, and failure reason from the Windows Security Event Log.
3. Counted the number of failed attempts and confirmed the rate exceeded the brute-force threshold within a short window.
4. Checked Active Directory Users and Computers to confirm the account lockout status.
5. Cross-referenced the source workstation against the lab asset inventory.
6. Reviewed Event ID 4624 (successful logon) entries around the same timeframe to determine if any attempt succeeded before lockout.
7. No successful logon was found in the review window.

---

## True Positive / False Positive Decision

**Decision: True Positive**

The pattern of failed logons met the criteria for a brute-force detection alert. The account lockout confirmed the threshold was reached. No successful logon occurred during the event window.

In a production environment, this would not be assumed to be a lab test and would require immediate escalation.

---

## Impact

| Area | Assessment |
|------|------------|
| Account status | Locked out after threshold exceeded |
| Successful compromise | No evidence of successful logon in the review window |
| Lateral movement | No lateral movement indicators observed during the event window |
| Detection coverage | Confirmed: Wazuh successfully ingested and alerted on Windows Security Event Log data |

---

## Recommended Action (Production Context)

In a production SOC environment, this alert would trigger:

1. Immediate review of the targeted account and confirmation of lockout status
2. Identification of the source workstation and review of its recent authentication history
3. Contact with the account owner to determine if they initiated the activity
4. If not authorized: isolation of the source workstation pending investigation
5. Review of other accounts targeted from the same source in the same window
6. Password reset for the targeted account after lockout investigation
7. Ticket creation with full timeline, source, target, and outcome documented

---

## Lessons Learned

- Windows Security Event Log provides specific, actionable data for authentication investigations. Event ID 4625 alone is not enough; logon type (2 for interactive, 3 for network) significantly affects the investigation direction.
- Wazuh thresholds must be tuned for the environment. A threshold that fires on two failed attempts produces noise; a threshold calibrated to the environment produces actionable alerts.
- Event ID 4740 (account lockout) paired with 4625 creates a cleaner detection chain than either event alone.
- Reviewing successful logon events (Event ID 4624) immediately after a brute-force alert is a critical step to determine if the attack succeeded.

---

## Screenshots / Redacted Evidence

Redacted screenshots available in the `evidence/` folder.

| File | Content |
|------|---------|
| `evidence/02-wazuh-auth-alert.png` | Wazuh alert showing failed logon pattern with rule details, account name redacted |
| `evidence/02-event-id-4625.png` | Windows Security Event Log showing 4625 entries, source IP and account name redacted |
| `evidence/02-account-lockout.png` | ADUC showing account lockout status, account name redacted |

*All screenshots are redacted to remove account names, hostnames, and internal IP addresses.*
