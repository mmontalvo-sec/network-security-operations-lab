# Detection Test Record

Use this template for each controlled detection scenario executed in the lab.

---

## Test Metadata

| Field | Value |
|---|---|
| Scenario name | |
| Date | YYYY-MM-DD |
| Start time | HH:MM |
| End time | HH:MM |
| Authorized by | Lab owner |
| Tester | |
| Lab phase | Phase 9 — Controlled Detection |

---

## Scope

| Field | Value |
|---|---|
| Source host / IP | |
| Destination host / IP | |
| Protocol / port | |
| Zone (source) | |
| Zone (destination) | |

---

## Objective

_What is this test designed to detect or validate?_

---

## Steps Taken

1.
2.
3.

---

## Expected Results

| Source | Expected Event |
|---|---|
| Windows Event Log | |
| Palo Alto Traffic Log | |
| Snort Alert Log | |
| Wazuh Alert | |

---

## Actual Results

| Source | Observed Event | Match? |
|---|---|---|
| Windows Event Log | | Yes / No / Partial |
| Palo Alto Traffic Log | | Yes / No / Partial |
| Snort Alert Log | | Yes / No / Partial |
| Wazuh Alert | | Yes / No / Partial |

---

## Evidence

| Item | Filename | Location |
|---|---|---|
| Screenshot 1 | | evidence/screenshots/ |
| Screenshot 2 | | evidence/screenshots/ |
| Log snippet | | evidence/sanitized-logs/ |

---

## Analyst Notes

_Observations, unexpected behavior, noise, or gaps._

---

## Recommendations

_Tuning actions, follow-up tasks, or policy changes._

---

## Outcome

- [ ] Detection confirmed
- [ ] Partial detection (gap noted)
- [ ] Not detected — investigation required
- [ ] Tuning needed
