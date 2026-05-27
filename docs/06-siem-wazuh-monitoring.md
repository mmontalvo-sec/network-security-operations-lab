# Wazuh SIEM — Monitoring and Log Collection

## Role in the Lab

Wazuh is the central log aggregation and analysis platform. All log sources — Windows
endpoints, the Palo Alto firewall, and Snort IDS — feed into Wazuh. It provides
dashboards, alert triage, event search, and investigation timeline capability.

Wazuh lives in VLAN 40 (172.16.40.0/24), isolated from the Windows server VLAN.
This separation is intentional: the SIEM is an analysis platform, not a domain service.

---

## Confirmed Deployment State

From `Wazuh_Agent_List.jpg` (May 12):

| Agent ID | Hostname | IP | OS | Status |
|---|---|---|---|---|
| 001 | client1 | 172.16.10.4 | Windows 10 Pro 10.0.19045.6456 | Active |
| 002 | client2 | 172.16.10.10 | Windows 10 Pro 10.0.19045.6456 | Active |
| 003 | redteam1 | 172.16.10.103 | Windows 10 Pro 10.0.19045.2965 | Active |
| 004 | wserverbackup | 172.16.0.8 | Windows Server 2016 Std Eval | Active |
| 005 | wserverdhcp | 172.16.0.7 | Windows Server 2016 Std Eval | Active |

**5 of 5 agents active. 0 disconnected.**

From `SIEM_alerts.jpg` (May 12, 14:02):
- 20,887 total events collected
- 0 Level 12+ critical alerts
- 8 authentication failures
- 233 authentication successes
- MITRE ATT&CK framework mapping active across multiple techniques

---

## Log Sources

| Source | Method | Events Collected |
|---|---|---|
| Windows clients (client1, client2) | Wazuh agent TCP 1514/1515 | Logon events, auth failures, object access |
| Windows servers (wserverbackup, wserverdhcp) | Wazuh agent TCP 1514/1515 | Auth, share access, file events |
| Palo Alto PA-500 | Syslog UDP 514 → Wazuh | Traffic logs, allowed/denied flows |
| Snort IDS | Wazuh agent localfile `/var/log/snort/alert` | IDS alerts, port scan detections |

---

## Key Windows Event IDs Collected

| Event ID | Description | Detection Value |
|---|---|---|
| 4624 | Successful logon | Authentication baseline |
| 4625 | Failed logon | Brute force / bad credentials |
| 4648 | Logon with explicit credentials | Lateral movement indicator |
| 4768 | Kerberos TGT request | Domain auth tracking |
| 4769 | Kerberos service ticket | Service access tracking |
| 5140 | Network share accessed | File share monitoring |
| 5145 | Share object access check | Granular share access |
| 4720 | User account created | Account lifecycle |
| 4726 | User account deleted | Account lifecycle |

---

## Wazuh Installation

Deployed using the official Wazuh installation assistant (single-node, all-in-one):

```bash
curl -sO https://packages.wazuh.com/4.x/wazuh-install.sh
sudo bash wazuh-install.sh -a
```

After install, access the Wazuh web UI at `https://<wazuh-ip>`.
Default credentials are displayed in the install output — change immediately.

---

## Agent Installation (Windows)

```powershell
# Download and install the Wazuh agent MSI
# Replace WAZUH_MANAGER with the Wazuh server IP (172.16.40.x)
msiexec /i wazuh-agent.msi WAZUH_MANAGER="172.16.40.10" /q

# Start the service
NET START WazuhSvc

# Verify agent is running
Get-Service WazuhSvc
```

---

## Palo Alto Syslog Configuration

In PAN-OS GUI:
1. Device → Server Profiles → Syslog → Add
   - Name: `wazuh-siem`
   - Server IP: `172.16.40.10`
   - Port: `514`
   - Transport: `UDP`
   - Format: `BSD`
2. Device → Log Settings → Traffic → Add profile → select `wazuh-siem`
3. Commit

---

## Snort Alert Forwarding to Wazuh

On the Snort Ubuntu host, edit the Wazuh agent config:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add this block:

```xml
<localfile>
  <log_format>snort-fast</log_format>
  <location>/var/log/snort/alert</location>
</localfile>
```

Restart the agent:

```bash
sudo systemctl restart wazuh-agent
```

---

## MITRE ATT&CK Coverage (Confirmed Active)

Wazuh dashboard showed active MITRE mapping during the demonstration. Techniques
detected during the 8 lab scenarios include:

| Technique | ID | Source |
|---|---|---|
| Network Service Discovery | T1046 | nmap scan from redteam1 |
| Brute Force | T1110 | SMB failed logins (4625) |
| Valid Accounts | T1078 | Successful logon after failures (4624) |
| SMB/Admin Shares | T1021.002 | Share access events (5140, 5145) |
| Data Manipulation | T1565 | File creation on monitored share |

---

## Investigation Workflow

```
1. Alert fires in Wazuh dashboard
2. Open Events tab → filter by rule ID or agent
3. Identify source IP, destination, username, timestamp
4. Cross-reference with other log sources:
   - Firewall: did this traffic cross a zone boundary?
   - Snort: was there a scan before the event?
   - Windows: what happened before and after?
5. Build timeline: earliest event → chain of events → final state
6. Document in incident-report-template.md
```
