# SOC Alert Walkthrough 05: False Positive Analysis

**Lab:** Network Security Operations Lab
**Date:** 2026-05
**Status:** Authorized lab activity
**Classification:** False Positive (documented and tuned)

---

## Alert Summary

| Field | Detail |
|-------|--------|
| Alert source | Wazuh SIEM |
| Alert type | Repeated authentication events flagged as suspicious |
| Trigger | Wazuh rule firing on normal domain authentication traffic |
| Source | Domain-joined Windows workstation during normal login |
| Severity | Medium (as initially alerted) |
| Decision | False Positive — rule tuned |

---

## Environment

| Component | Detail |
|-----------|--------|
| SIEM | Wazuh, agents on Windows Server and workstation |
| Source activity | Normal user login to domain-joined workstation |
| Log source | Windows Security Event Log (Event ID 4624, 4768, 4769) |
| Network | VLAN-segmented lab with Active Directory domain |

---

## Trigger

After configuring Wazuh to ingest Windows Security Event Logs, the SIEM began generating repeated medium-severity alerts during normal user logins. The rule was firing on Kerberos authentication events (Event ID 4768 and 4769) that occur as part of every standard domain login process.

---

## Data Sources

| Source | What It Provided |
|--------|-----------------|
| Wazuh SIEM | Alert details and rule ID triggering the false positive |
| Windows Security Event Log | Raw authentication events showing normal login flow |
| Active Directory | Confirmation that the user account was valid and login was expected |

---

## Observed Evidence

Wazuh was generating medium-severity alerts with rule language suggesting suspicious authentication activity. On investigation, the raw events showed:

| Event ID | Meaning | Context |
|----------|---------|---------|
| 4768 | Kerberos Authentication Service request | Normal domain login |
| 4769 | Kerberos Service Ticket request | Normal domain resource access |
| 4624 | Successful logon | User logged in successfully |

The pattern was entirely consistent with a standard domain user logging into a domain-joined workstation and accessing shared resources. No anomalous behavior was present.

The Wazuh rule was tuned too broadly and was treating normal Kerberos activity as suspicious based on event frequency rather than behavioral context.

---

## Triage Steps

1. Reviewed the Wazuh alert and noted the rule ID and triggering event IDs.
2. Pulled the raw Windows Security Event Log entries for the alerting window.
3. Identified the user account and workstation involved.
4. Confirmed the user was a known account in Active Directory with no recent changes.
5. Reviewed the login time and compared against expected usage patterns for the lab.
6. Confirmed no associated failed logons, lateral movement, or anomalous network activity in the same window.
7. Concluded the alert was a false positive caused by broad rule configuration.

---

## True Positive / False Positive Decision

**Decision: False Positive**

The activity was normal domain authentication. The Wazuh rule was firing on standard Kerberos ticket requests without accounting for the baseline volume expected in an Active Directory environment. No suspicious behavior was present.

---

## Tuning Action Taken

Reviewed the Wazuh rule triggering the alert and added context to narrow the detection scope. Updated the rule to require additional indicators beyond event frequency before triggering at medium severity. Re-tested with normal login activity to confirm the false positive no longer fired.

Documented the tuning decision and the baseline Kerberos event volume for the lab environment to support future alert calibration.

---

## Impact

| Area | Assessment |
|------|------------|
| Alert noise | Reduced after tuning |
| Detection coverage | Maintained for actual anomalous Kerberos patterns |
| Baseline documentation | Established normal login event volume for this environment |

---

## Recommended Action (Production Context)

False positive management is a continuous process in any SOC environment:

1. Document each false positive with the rule ID, triggering event, and decision rationale
2. Tune rules to reduce noise without eliminating coverage for real threats
3. Build a baseline of normal authentication volume before setting thresholds
4. Track false positive rate over time to measure detection quality improvement
5. Never delete a rule because of a false positive; tune it

---

## Lessons Learned

- False positives are not failures. They are part of the calibration process. A SIEM that generates no false positives is probably also missing real events.
- Kerberos authentication events require baseline context to be useful. Raw event volume alone is not a reliable indicator of suspicious behavior in an Active Directory environment.
- Documenting false positives is as valuable as documenting true positives. Both inform how the detection environment should be tuned over time.
- The difference between a junior analyst and a useful analyst is the ability to distinguish a real alert from a noisy rule without needing someone else to tell them which is which.

---

## Screenshots / Redacted Evidence

| File | Content |
|------|---------|
| `evidence/05-wazuh-fp-alert.png` | Wazuh alert showing false positive rule trigger, account details redacted |
| `evidence/05-event-log-kerberos.png` | Windows Security Event Log showing normal 4768/4769 pattern, account and host redacted |
| `evidence/05-rule-tuning-notes.png` | Wazuh rule modification notes |
