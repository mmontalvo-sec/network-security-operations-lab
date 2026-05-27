# Active Directory and Windows Infrastructure

## Overview

The Windows infrastructure is the identity core of the lab. Every authentication event,
DNS query, DHCP lease, and domain logon flows through it — and every one of those events
is a log source for the Wazuh SIEM.

Three Windows Server 2016 hosts were deployed, all domain-joined to the `soc.lab` domain.
Two Windows 10 clients were also joined to the domain. All five hosts run Wazuh agents.

---

## Server Inventory

| Hostname | IP | Role | VLAN |
|---|---|---|---|
| DC01 | 172.16.0.6 | Primary DC, DNS, NTP | 20 |
| wserverdhcp | 172.16.0.7 | DHCP server | 20 |
| wserverbackup | 172.16.0.8 | Backup / file share server | 20 |
| client1 | 172.16.10.4 | Domain-joined workstation | 10 |
| client2 | 172.16.10.10 | Domain-joined workstation | 10 |

---

## Domain Configuration

| Setting | Value |
|---|---|
| Domain name (FQDN) | soc.lab |
| NetBIOS name | SOC |
| Forest functional level | Windows Server 2016 |
| Domain controller | DC01 |
| DNS server | DC01 (172.16.0.6) |
| NTP source | DC01 (domain time hierarchy) |

---

## Phase 1 — Server Preparation

Before installing AD DS, the server must be fully prepared.

### Static IP Configuration

On DC01 (Windows Server 2016), set a static IP before promoting to DC:

```
IP Address:      172.16.0.6
Subnet Mask:     255.255.255.0
Default Gateway: 172.16.20.1  (Palo Alto subinterface ethernet1/2.20)
DNS Server:      127.0.0.1    (points to itself after DNS is installed)
```

### Rename the Server

```powershell
Rename-Computer -NewName "DC01" -Restart
```

### Verify Basic Reachability

Before installing any roles:

```cmd
ping 172.16.20.1       # gateway (Palo Alto)
ping 172.16.10.1       # client VLAN gateway (cross-VLAN test)
ipconfig /all
```

Do not proceed if the gateway is unreachable. A network problem at this stage
will break AD promotion later and is much harder to troubleshoot after the fact.

---

## Phase 2 — Install AD DS and DNS

### Install Roles

```powershell
Install-WindowsFeature AD-Domain-Services,DNS -IncludeManagementTools
```

### Promote to Domain Controller

```powershell
Install-ADDSForest `
    -DomainName "soc.lab" `
    -DomainNetbiosName "SOC" `
    -InstallDNS `
    -Force
```

The server restarts automatically after promotion completes.

### Post-Reboot Validation

```powershell
Get-ADDomain
Get-ADForest
ipconfig /all
nslookup dc01.soc.lab
```

Expected results:
- Domain: `soc.lab` confirmed
- Forest: `soc.lab` confirmed
- DC01 resolves by FQDN via DNS

---

## Phase 3 — Organizational Unit Structure

Created in Active Directory Users and Computers (ADUC):

```
soc.lab
├── Servers        ← domain-joined servers (DC01, wserverdhcp, wserverbackup)
├── Workstations   ← domain-joined clients (client1, client2)
├── Users          ← standard user accounts
└── Admins         ← admin accounts (separate from daily-use accounts)
```

### Users Created

| Username | OU | Purpose |
|---|---|---|
| aduser1 | Users | Standard domain user — demo scenarios |
| aduser2 | Users | Standard domain user — demo scenarios |
| aduser3 | Users | Standard domain user — demo scenarios |

```powershell
# Create OUs
New-ADOrganizationalUnit -Name "Servers"      -Path "DC=soc,DC=lab"
New-ADOrganizationalUnit -Name "Workstations" -Path "DC=soc,DC=lab"
New-ADOrganizationalUnit -Name "Users"        -Path "DC=soc,DC=lab"
New-ADOrganizationalUnit -Name "Admins"       -Path "DC=soc,DC=lab"

# Create demo users
New-ADUser -Name "aduser1" -SamAccountName "aduser1" `
    -UserPrincipalName "aduser1@soc.lab" `
    -Path "OU=Users,DC=soc,DC=lab" `
    -AccountPassword (ConvertTo-SecureString "LabPass123!" -AsPlainText -Force) `
    -Enabled $true

New-ADUser -Name "aduser2" -SamAccountName "aduser2" `
    -UserPrincipalName "aduser2@soc.lab" `
    -Path "OU=Users,DC=soc,DC=lab" `
    -AccountPassword (ConvertTo-SecureString "LabPass123!" -AsPlainText -Force) `
    -Enabled $true

New-ADUser -Name "aduser3" -SamAccountName "aduser3" `
    -UserPrincipalName "aduser3@soc.lab" `
    -Path "OU=Users,DC=soc,DC=lab" `
    -AccountPassword (ConvertTo-SecureString "LabPass123!" -AsPlainText -Force) `
    -Enabled $true
```

---

## Phase 4 — DNS Validation

DNS is installed automatically with AD DS. Verify it is working correctly.

```cmd
# On DC01
nslookup dc01
nslookup dc01.soc.lab
ping dc01.soc.lab

# From a client (set DNS to 172.16.0.6 first)
nslookup dc01.soc.lab 172.16.0.6
```

In DNS Manager, confirm:
- Forward lookup zone: `soc.lab` exists
- A record: `dc01.soc.lab → 172.16.0.6`
- Reverse lookup zone (optional but recommended)

**Common failure:** Client cannot resolve `soc.lab` but DC can.
Root cause is usually a firewall rule blocking CLIENTS → WINDOWS-SERVERS on port 53 (DNS).
Check the Palo Alto traffic log for denied traffic from VLAN 10 to VLAN 20.

---

## Phase 5 — NTP Configuration

DC01 is the NTP reference for the entire lab. Consistent time is mandatory for log
correlation — a 5-minute clock skew can make an event timeline unreadable.

```cmd
# On DC01 - sync to external source (temporarily during setup)
w32tm /config /manualpeerlist:"pool.ntp.org" /syncfromflags:manual /reliable:YES /update
net stop w32tm && net start w32tm
w32tm /resync /force

# Verify DC01 time source
w32tm /query /status
w32tm /query /peers
```

Domain-joined clients sync to DC01 automatically via the Windows Time Service.
Non-domain devices (Snort, Wazuh) should also point to `172.16.0.6` as their NTP server.

---

## Phase 6 — Domain Join (Clients and Servers)

### Join a Windows 10 Client

```powershell
# Set DNS to point to DC01 first
Add-Computer -DomainName "soc.lab" `
    -Credential (Get-Credential) `
    -Restart
```

Or via GUI: System Properties → Computer Name → Change → Domain → `soc.lab`

### Validation After Join

```powershell
# On the joined client
Get-ADComputer $env:COMPUTERNAME
whoami /fqdn
nslookup dc01.soc.lab
```

Confirm in ADUC on DC01 that the computer object appears under the Workstations OU.

### All Joined Hosts (Confirmed)

| Host | Type | Wazuh Agent ID | Status |
|---|---|---|---|
| client1 | Windows 10 Pro | 001 | Active |
| client2 | Windows 10 Pro | 002 | Active |
| redteam1 | Windows 10 Pro | 003 | Active |
| wserverbackup | Windows Server 2016 | 004 | Active |
| wserverdhcp | Windows Server 2016 | 005 | Active |

---

## Phase 7 — DHCP Configuration

DHCP is hosted on `wserverdhcp` (172.16.0.7), not on the firewall.
The Palo Alto acts as a DHCP relay agent — it forwards DHCP broadcasts from each VLAN
to the Windows DHCP server.

### Install DHCP Role on wserverdhcp

```powershell
Install-WindowsFeature DHCP -IncludeManagementTools
```

### Create DHCP Scope for VLAN 10 (Clients)

```powershell
Add-DhcpServerv4Scope `
    -Name "CLIENTS-VLAN10" `
    -StartRange 172.16.10.100 `
    -EndRange 172.16.10.200 `
    -SubnetMask 255.255.255.0 `
    -State Active

Set-DhcpServerv4OptionValue -ScopeId 172.16.10.0 `
    -Router 172.16.10.1 `
    -DnsServer 172.16.0.6 `
    -DnsDomain "soc.lab"
```

### DHCP Relay on Palo Alto

Configure DHCP relay on each subinterface in the Palo Alto web GUI:
- Network → DHCP → DHCP Relay
- Interface: ethernet1/2.10 (and other VLANs as needed)
- Server IP: 172.16.0.7 (wserverdhcp)

---

## Phase 8 — Shared Folder (wserverbackup)

A public shared folder was created on `wserverbackup` for demo scenario use.

```
Share path: \\wserverbackup\Share
Access: domain users
Purpose: SMB access detection demo (Events 5140, 5145)
```

File access to this share generates Windows audit events that Wazuh agents forward
to the SIEM, producing detectable events during the demonstration scenarios.

---

## Windows Event IDs — Log Value Reference

| Event ID | Description | Detection Value |
|---|---|---|
| 4624 | Successful logon | Baseline — logon type and source IP |
| 4625 | Failed logon | Authentication failure — brute force detection |
| 4648 | Logon with explicit credentials | Lateral movement indicator |
| 4768 | Kerberos TGT requested | Domain authentication baseline |
| 4769 | Kerberos service ticket requested | Service access tracking |
| 5140 | Network share was accessed | File share access monitoring |
| 5145 | Network share object access check | Granular share access |
| 4720 | User account created | Account creation tracking |
| 4726 | User account deleted | Account deletion tracking |

All of the above are forwarded to Wazuh via the agent installed on each Windows host.

---

## Evidence to Capture

- [ ] ADUC screenshot showing OU structure (Servers, Workstations, Users, Admins)
- [ ] ADUC screenshot showing aduser1, aduser2, aduser3 in Users OU
- [ ] Computer objects in ADUC (client1, client2, wserverbackup, wserverdhcp)
- [ ] DNS Manager showing `soc.lab` forward lookup zone
- [ ] `nslookup dc01.soc.lab` output
- [ ] `w32tm /query /status` on DC01
- [ ] Domain join confirmation on a client (System Properties)
- [ ] DHCP scope screenshot on wserverdhcp
- [ ] DHCP active leases screenshot
