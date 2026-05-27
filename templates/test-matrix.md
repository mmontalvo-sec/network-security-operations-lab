# Test Matrix Template

## Test Session

| Field | Value |
|---|---|
| Date | YYYY-MM-DD |
| Tester |  |
| Phase |  |
| Lab Status |  |

## Matrix

| ID | Test | Source | Destination | Expected Result | Actual Result | Evidence | Pass/Fail |
|---|---|---|---|---|---|---|---|
| T-001 | Gateway reachability | Client VLAN 10 | 172.16.10.1 | Ping succeeds |  |  |  |
| T-002 | DNS resolution | Client | dc01.soc.lab | Resolves to DC IP |  |  |  |
| T-003 | Domain login | Domain client | DC01 | Login succeeds |  |  |  |
| T-004 | Firewall logging | VLAN 10 | VLAN 20 | Traffic log visible |  |  |  |
| T-005 | Snort visibility | Test host | DC01 | Snort sees traffic |  |  |  |
| T-006 | Wazuh agent | DC01 | Wazuh | Agent connected |  |  |  |
| T-007 | Palo Alto syslog | Firewall | Wazuh | Syslog visible |  |  |  |
| T-008 | Failed login alert | Client | DC01 | Failed logon event visible |  |  |  |
| T-009 | Safe scan alert | Test host | DC01 | IDS/SIEM alert visible |  |  |  |

## Summary

- Passed:
- Failed:
- Blocked:

## Follow-Up Actions

| Action | Owner | Priority | Status |
|---|---|---|---|
|  |  |  |  |
