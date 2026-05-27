# Detection and Validation Scenarios

## Overview

These scenarios were designed as structured demonstrations of the lab's detection and monitoring capability. They were executed during the final lab demonstration using the Wazuh SIEM, Snort IDS, Windows event logs, and Palo Alto firewall logs as evidence sources.

All activity was performed inside the owned, isolated lab environment only. Kali Linux was installed via VirtualBox on red team hosts. The redteam1 host (172.16.10.103) was placed in VLAN 10 during the demonstration to simplify network access.

---

## Confirmed Active Environment at Time of Demo

- Wazuh: 5 active agents (client1, client2, redteam1, wserverbackup, wserverdhcp)
- Total events: 20,887
- Authentication failures: 8
- Authentication successes: 233
- MITRE ATT&CK mapping: active on dashboard

---

## Scenario 1 — Confirm Active Wazuh Agents

**Objective:** Verify that all enrolled endpoints are actively reporting to Wazuh.

**Systems:** Wazuh manager, all enrolled agents

**Steps:**
1. Navigate to Wazuh → Server Management → Endpoints Summary
2. Confirm agent status, IP, OS, and last keep-alive

**Result confirmed:** 5 agents active. 0 disconnected.

**What this proves:** The SIEM collection pipeline is working across Windows clients and servers.

**Evidence:** `Wazuh_Agent_List.jpg` — 5 agents, all active.

---

## Scenario 2 — Nmap Ping Sweep (Authorized Reconnaissance)

**Objective:** Generate basic network reconnaissance traffic and confirm visibility in Wazuh and/or Snort.

**Systems:** redteam1 (Kali) → 172.16.20.0/24 or 172.16.10.0/24

**Steps (from Kali):**
```bash
ip a
nmap -sn 172.16.20.0/24
# or
nmap -sn 172.16.10.0/24
```

**Expected detections:**
- Wazuh: events from scanned hosts showing ICMP or ARP activity
- Snort: ICMP sweep detection (if ICMP rules enabled)
- Firewall: traffic from redteam1 IP crossing zone boundary (if inter-zone)

**What this proves:**
- Basic recon traffic is visible in the environment
- The red team host generates detectable activity
- MITRE ATT&CK: T1046 (Network Service Discovery), T1018 (Remote System Discovery)

---

## Scenario 3 — Nmap Port Scan Against Windows Server

**Objective:** Detect service enumeration against the primary Windows server.

**Systems:** redteam1 (Kali) → 172.16.0.6

**Steps:**
```bash
nmap -p 53,88,135,139,389,445,3389 172.16.0.6
# or
nmap -sT 172.16.0.6
# or (requires sudo)
sudo nmap -sS 172.16.0.6
```

**Expected detections:**
- Wazuh: network events from target host
- Snort: port scan detection rules fire
- Firewall Traffic Monitor: traffic from redteam1 to Windows server VLAN

**What this proves:**
- Service enumeration targeting AD, Kerberos, LDAP, SMB, and RDP is detectable
- Multi-source correlation: nmap → Snort → firewall → Wazuh
- MITRE ATT&CK: T1046 (Network Service Scanning)

---

## Scenario 4 — SMB Failed Authentication (Brute-Force Simulation)

**Objective:** Generate Windows authentication failure events and detect them in Wazuh.

**Systems:** redteam1 (Kali) → 172.16.0.6

**Steps:**
```bash
smbclient -L //172.16.0.6 -U fakeuser
# Enter wrong password — repeat 3 times

smbclient -L //172.16.0.6 -U testuser
# Enter wrong password — repeat
```

**Expected detections:**
- Windows Event ID 4625 — Failed logon (appears on domain controller and target server)
- Wazuh: alert for authentication failure with source IP, username, and timestamp
- Multiple failures from same source IP visible in Wazuh search

**What this proves:**
- Failed authentication is collected from Windows hosts via Wazuh agent
- Brute-force patterns (repeated failures, same source) are visible in SIEM
- MITRE ATT&CK: T1110 (Brute Force)

**Evidence:** Wazuh dashboard shows 8 authentication failures.

---

## Scenario 5 — Successful Login After Failed Attempts (Timeline Correlation)

**Objective:** Demonstrate why event correlation matters — a successful login after multiple failures is more significant than a normal login in isolation.

**Systems:** Domain user on client or server

**Steps:**
1. Complete Scenario 4 (failed login attempts)
2. Log in correctly with a valid domain account via RDP, SMB, or interactive login
3. Search Wazuh for Event ID 4624 for the same user and timeframe

**Expected detections:**
- Windows Event ID 4624 — Successful logon (Logon Type 2, 3, or 10)
- Wazuh: 4624 alert with username, source IP, logon type
- Timeline view: failures (4625) → success (4624) visible in sequence

**What this proves:**
- Wazuh can correlate authentication events across time
- A success-after-failure pattern is a key SOC investigation starting point
- MITRE ATT&CK: T1078 (Valid Accounts)

**Evidence:** Wazuh dashboard shows 233 authentication successes — baseline established.

---

## Scenario 6 — SMB Shared Folder Access

**Objective:** Detect file share access using Windows audit events.

**Systems:** client1 or client2 → `\\wserverbackup\Share`

**Steps:**
1. Open File Explorer → `\\wserverbackup\Share`
2. Browse share contents
3. Create a file: `demo-alert.txt`
4. Modify and save the file

**In Wazuh search for:**
- `5140` — A network share object was accessed
- `5145` — A network share object was checked for access

**What this proves:**
- File share access is audited at the Windows level
- Wazuh agents forward object access events to the SIEM
- Source IP, username, share name, and timestamp visible in log
- MITRE ATT&CK: T1021.002 (SMB/Windows Admin Shares)

---

## Scenario 7 — File Integrity Monitoring (FIM)

**Objective:** Detect file creation or modification on a monitored Windows host.

**Systems:** Domain client → wserverbackup shared folder

**Steps:**
1. Create `demo-alert.txt` on the shared folder (from Scenario 6)
2. Modify the file
3. Search Wazuh for syscheck events related to the file

**Expected detection:**
- Wazuh syscheck (FIM) alert: file created or modified on monitored path
- File name, hash, modification time visible in alert

**What this proves:**
- Wazuh FIM monitors filesystem changes on enrolled hosts
- File creation events on server shares can be detected and investigated
- MITRE ATT&CK: T1565 (Data Manipulation)

---

## Scenario 8 — Wazuh Agent Validation Baseline

**Objective:** Confirm that all agents are active and contributing to the SIEM before beginning adversarial scenarios.

**Steps:**
1. Navigate to Wazuh → Endpoints Summary
2. Record: agent names, IPs, OS, status, last keep-alive
3. Confirm all 5 are active

**What this proves:**
- SIEM collection is working across all enrolled endpoints
- This is the baseline from which all anomaly detection is measured

---

## Investigation Timeline Template

For any scenario that produces detectable events, the following timeline structure should be completed:

```
Time        Source          Event                               Notes
---------   ----------      ----------------------------------  --------------------
HH:MM:SS    redteam1        nmap -sS 172.16.0.6              Scan initiated
HH:MM:SS    Snort           Port scan detected                  Alert in /var/log/snort/alert
HH:MM:SS    172.16.0.6    Event 4625 — Failed SMB logon       fakeuser, src 172.16.10.103
HH:MM:SS    172.16.0.6    Event 4625 — Failed SMB logon       fakeuser, repeat
HH:MM:SS    172.16.0.6    Event 4624 — Successful logon       testuser, Type 3
HH:MM:SS    wserverbackup   Event 5140 — Share accessed         \\wserverbackup\Share
HH:MM:SS    wserverbackup   Event 5145 — Object checked         demo-alert.txt
HH:MM:SS    Wazuh           FIM alert — file created            demo-alert.txt hash logged
```

This timeline structure is what a SOC analyst builds during investigation and what an IR report documents as findings.
