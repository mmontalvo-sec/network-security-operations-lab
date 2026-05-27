# Runbook: Controlled Detection Scenarios

## Purpose

This runbook defines the detection scenarios designed and executed during the Network Security Operations Lab demonstration. All scenarios are authorized, scoped to the owned lab environment, and documented with expected log events and evidence requirements.

The scenarios were designed to demonstrate the complete SOC monitoring pipeline: endpoint activity → Windows event log → Wazuh agent → Wazuh manager → alert dashboard → investigation timeline.

---

## Prerequisites

- 5 Wazuh agents enrolled and confirmed active
- Palo Alto firewall online with inter-zone logging enabled
- Snort IDS running on Ubuntu sensor with SPAN traffic confirmed
- Windows Server shared folder configured: `\\wserverbackup\Share`
- AD user accounts: aduser1, aduser2, aduser3 created
- Kali Linux available on redteam1 (172.16.10.103) via VirtualBox
- Wazuh manager accessible

---

## Scenario 1 — View Active Wazuh Agents

**Objective:** Confirm SIEM collection is working before running any detection scenarios.

**Steps:**
```
Wazuh → Server Management → Endpoints Summary
```

**Confirm:**
- Agent Name visible
- IP address correct
- Status: Active
- Last keep-alive: recent
- OS identified

**Expected:** 5 agents active (client1, client2, redteam1, wserverbackup, wserverdhcp)

**Evidence:** Screenshot of Agents page showing all active.

---

## Scenario 2 — Nmap Ping Sweep

**Objective:** Generate reconnaissance traffic and observe SIEM/IDS visibility.

**From Kali on redteam1:**
```bash
ip a
nmap -sn 172.16.20.0/24
nmap -sn 172.16.10.0/24
```

**In Wazuh search for:**
```
nmap
scan
172.16.10.103  (redteam1 source IP)
```

Also check Snort logs if integrated.

**Evidence to capture:**
- Kali terminal showing nmap execution
- Wazuh event showing related activity
- Source IP and timestamp visible

---

## Scenario 3 — Nmap Port Scan Against Windows Server

**Objective:** Detect service discovery targeting the domain controller/server.

**From Kali on redteam1:**
```bash
nmap -p 53,88,135,139,389,445,3389 172.16.0.6
nmap -sT 172.16.0.6
sudo nmap -sS 172.16.0.6
```

**In Wazuh search for:**
```
scan
nmap
172.16.0.6
172.16.10.103
```

If Snort is integrated, search for port scan alerts.

**Evidence to capture:**
- nmap output showing open ports
- Wazuh/Snort event
- Source IP, destination IP, ports detected

---

## Scenario 4 — SMB Failed Authentication

**Objective:** Generate Windows Event ID 4625 (failed logon) events.

**From Kali on redteam1:**
```bash
smbclient -L //172.16.0.6 -U fakeuser
# Enter wrong password when prompted
# Repeat 3 times

smbclient -L //172.16.0.6 -U aduser1
# Enter wrong password when prompted
```

**In Wazuh search for:**
```
4625
failed logon
fakeuser
aduser1
```

**Expected event:** Windows Event ID 4625 — Failed Logon  
Visible on the target server and forwarded to Wazuh.

**Evidence to capture:**
- Kali terminal showing smbclient attempt
- Wazuh alert showing Event 4625
- Username, source IP, target host, timestamp

---

## Scenario 5 — Successful Login After Failed Attempts

**Objective:** Show that a successful login following failures is detectable and correlatable.

**Steps:**
1. Complete Scenario 4 first
2. Log in with a valid domain account:
   - Via RDP to a Windows client
   - Via SMB: `smbclient -L //172.16.0.6 -U aduser1` with correct password
   - Via interactive domain login on a client workstation

**In Wazuh search for:**
```
4624
successful logon
aduser1
```

**Expected event:** Windows Event ID 4624 — Successful Logon

**Evidence to capture:**
- Event 4624 in Wazuh
- Username, host, logon type, timestamp
- Show the timeline: 4625 events → 4624 event in sequence

---

## Scenario 6 — SMB Shared Folder Access

**Prerequisite:** Shared folder exists on wserverbackup: `\\wserverbackup\Share`

**From a domain Windows client:**
```
Open File Explorer
Navigate to \\wserverbackup\Share
Browse contents
Create new text file: demo-alert.txt
Modify content and save
```

**In Wazuh search for:**
```
5140
5145
Share
demo-alert.txt
```

**Expected events:**
- 5140 — A network share object was accessed
- 5145 — A network share object was checked for access

**Evidence to capture:**
- File Explorer showing share access
- Wazuh showing 5140 and 5145 events
- Share name, username, source IP, timestamp

---

## Scenario 7 — File Integrity Monitoring Alert

**Objective:** Trigger a Wazuh FIM (syscheck) alert through file creation or modification.

**Steps:**
1. Create `demo-alert.txt` on the share or a monitored local path (from Scenario 6)
2. Modify and save the file
3. Wait for syscheck cycle (usually 6–12 hours by default; can be shortened in ossec.conf)

**In Wazuh search for:**
```
syscheck
FIM
demo-alert.txt
550  (syscheck file added)
554  (syscheck file modified)
```

**Evidence to capture:**
- Wazuh syscheck alert showing file path, hash, modification time
- Agent source identified

---

## Scenario 8 — Wazuh Dashboard Review and Timeline

**Objective:** Use Wazuh as an investigation tool to build an event timeline from the completed scenarios.

**Steps:**
1. Navigate to Wazuh → Threat Monitoring dashboard
2. Set time range to cover the demo window
3. Review:
   - Total event count
   - Authentication failure events
   - MITRE ATT&CK technique distribution
   - Top agents contributing
4. Use Events tab → search for specific Event IDs
5. Filter by agent name to trace activity per host
6. Export or screenshot the timeline for the IR documentation

**Evidence to capture:**
- Dashboard screenshot showing total events, auth failures, auth successes
- MITRE ATT&CK chart
- Events table filtered to demo timeframe
- Timeline showing scenario sequence

---

## Notes for Documentation

Each completed scenario should be recorded using the `detection-test-template.md` in the `templates/` directory.

Store completed records in `evidence/validation-results/`.

Screenshots go in `evidence/screenshots/` using the naming convention:
```
scenario-[N]-[short-description]-[YYYY-MM-DD].png
```

Example:
```
scenario-04-smb-failed-auth-2026-05-12.png
scenario-05-successful-login-correlation-2026-05-12.png
```
