# Runbook: Windows Server — AD DS, DNS, DHCP, NTP, Domain Join

## Purpose

Deploy and validate the Windows identity core of the lab:
- Active Directory Domain Services on DC01
- DNS integrated with AD
- NTP via Windows Time Service
- DHCP on a dedicated server (wserverdhcp)
- Domain join for all client and server hosts

## Prerequisites

- DC01 has a static IP (172.16.0.6) and is reachable from VLAN 20
- Palo Alto firewall allows CLIENTS → WINDOWS-SERVERS traffic (DNS port 53, Kerberos 88, LDAP 389)
- BlueSwitch has DC01 on a VLAN 20 access port

---

## Step 1 — Prepare DC01

```powershell
# Rename server
Rename-Computer -NewName "DC01" -Restart

# After reboot — verify networking
ping 172.16.20.1       # Palo Alto gateway
ipconfig /all          # Confirm static IP, mask, gateway, DNS
```

Static IP settings:
- IP: 172.16.0.6
- Mask: 255.255.255.0
- Gateway: 172.16.20.1
- DNS: 127.0.0.1 (after DNS role installed)

---

## Step 2 — Install Roles and Promote DC

```powershell
# Install AD DS and DNS roles
Install-WindowsFeature AD-Domain-Services,DNS -IncludeManagementTools

# Promote to Domain Controller — creates new forest
Install-ADDSForest `
    -DomainName "soc.lab" `
    -DomainNetbiosName "SOC" `
    -InstallDNS `
    -Force
```

Server restarts automatically. After reboot:

```powershell
Get-ADDomain
Get-ADForest
nslookup dc01.soc.lab
```

---

## Step 3 — Create OU Structure and Users

```powershell
# Create OUs
New-ADOrganizationalUnit -Name "Servers"      -Path "DC=soc,DC=lab"
New-ADOrganizationalUnit -Name "Workstations" -Path "DC=soc,DC=lab"
New-ADOrganizationalUnit -Name "Users"        -Path "DC=soc,DC=lab"
New-ADOrganizationalUnit -Name "Admins"       -Path "DC=soc,DC=lab"

# Create lab users (use strong passwords in production)
$pw = ConvertTo-SecureString "LabPass123!" -AsPlainText -Force

New-ADUser -Name "aduser1" -SamAccountName "aduser1" `
    -UserPrincipalName "aduser1@soc.lab" `
    -Path "OU=Users,DC=soc,DC=lab" `
    -AccountPassword $pw -Enabled $true

New-ADUser -Name "aduser2" -SamAccountName "aduser2" `
    -UserPrincipalName "aduser2@soc.lab" `
    -Path "OU=Users,DC=soc,DC=lab" `
    -AccountPassword $pw -Enabled $true

New-ADUser -Name "aduser3" -SamAccountName "aduser3" `
    -UserPrincipalName "aduser3@soc.lab" `
    -Path "OU=Users,DC=soc,DC=lab" `
    -AccountPassword $pw -Enabled $true
```

Validate in ADUC: confirm all OUs and users are visible.

---

## Step 4 — Configure NTP

```cmd
w32tm /config /manualpeerlist:"pool.ntp.org" /syncfromflags:manual /reliable:YES /update
net stop w32tm && net start w32tm
w32tm /resync /force
w32tm /query /status
```

Expected: `Source: pool.ntp.org` or `Source: Local CMOS Clock` (acceptable in lab).
Domain members sync to DC01 automatically.
Non-domain hosts (Wazuh, Snort) should use 172.16.0.6 as their NTP server.

---

## Step 5 — Install DHCP on wserverdhcp

```powershell
Install-WindowsFeature DHCP -IncludeManagementTools

# Authorize DHCP server in AD
Add-DhcpServerInDC -DnsName "wserverdhcp.soc.lab" -IPAddress 172.16.0.7

# Create scope for VLAN 10 Clients
Add-DhcpServerv4Scope `
    -Name "CLIENTS-VLAN10" `
    -StartRange 172.16.10.100 `
    -EndRange 172.16.10.200 `
    -SubnetMask 255.255.255.0 `
    -State Active

# Set scope options
Set-DhcpServerv4OptionValue -ScopeId 172.16.10.0 `
    -Router 172.16.10.1 `
    -DnsServer 172.16.0.6 `
    -DnsDomain "soc.lab"
```

Configure relay on Palo Alto:
Network → DHCP → DHCP Relay → ethernet1/2.10 → Server: 172.16.0.7

---

## Step 6 — Domain Join Clients and Servers

```powershell
# On each client or server to be joined
Add-Computer -DomainName "soc.lab" `
    -Credential (Get-Credential) `
    -OUPath "OU=Workstations,DC=soc,DC=lab" `
    -Restart
```

Or via GUI: Control Panel → System → Change Settings → Domain → `soc.lab`

After joining, move computer objects to the correct OU in ADUC if needed.

---

## Step 7 — Configure Shared Folder on wserverbackup

```powershell
# Create share
New-Item -Path "C:\Share" -ItemType Directory
New-SmbShare -Name "Share" -Path "C:\Share" `
    -FullAccess "SOC\Domain Users"
```

This share is used in detection scenarios 6 and 7 to generate Events 5140 and 5145.

---

## Validation Commands

```powershell
# Domain and forest
Get-ADDomain
Get-ADForest
Get-ADComputer -Filter *
Get-ADUser -Filter *

# DNS
nslookup dc01.soc.lab
nslookup soc.lab

# NTP
w32tm /query /status
w32tm /query /peers

# DHCP leases
Get-DhcpServerv4Lease -ScopeId 172.16.10.0

# Domain membership on client
(Get-WmiObject Win32_ComputerSystem).Domain
whoami /fqdn
```

---

## Expected Results

| Check | Expected |
|---|---|
| `Get-ADDomain` | `soc.lab` returned |
| `nslookup dc01.soc.lab` | Returns 172.16.0.6 |
| `w32tm /query /status` | Source set, no errors |
| ADUC | Servers, Workstations, Users, Admins OUs present |
| ADUC | aduser1, aduser2, aduser3 in Users OU |
| ADUC | client1, client2, wserverbackup, wserverdhcp in correct OUs |
| DHCP | Active leases visible for VLAN 10 clients |
| Wazuh | All 5 agents active |

---

## Troubleshooting

| Problem | Likely Cause | Fix |
|---|---|---|
| AD promotion fails | Static IP not set, or wrong DNS | Set IP/DNS before promoting |
| Client can't resolve soc.lab | DNS not set to 172.16.0.6 | Set DNS server in NIC settings |
| Client can't resolve across VLANs | Firewall blocking UDP/TCP 53 | Add CLIENTS→WINDOWS-SERVERS allow rule for DNS |
| Domain join fails | Client can't reach DC | Check routing, firewall, DNS |
| DHCP not assigning leases | Relay not configured on firewall | Configure relay on Palo Alto subinterfaces |
| NTP skew issues | DC01 not syncing | Run w32tm /resync /force on DC01 |
| Wazuh agent auth fails | Time skew >5 min | Fix NTP, restart agent |
