# Runbook: Windows Server AD DS, DNS, NTP, and DHCP

## Goal

Deploy Windows Server 2016 as the identity and infrastructure core for the lab.

## Server Plan

| Setting | Value |
|---|---|
| Hostname | DC01 |
| VLAN | 20 |
| IP | 172.16.20.10 |
| Gateway | 172.16.20.1 |
| DNS | 172.16.20.10 |
| Domain | soc.lab |

## Step 1: Configure Static IP

Run on Windows Server using GUI or PowerShell.

PowerShell example:

```powershell
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 172.16.20.10 -PrefixLength 24 -DefaultGateway 172.16.20.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 172.16.20.10
```

Adjust `InterfaceAlias` if your adapter name is different.

## Step 2: Rename Server

Run in PowerShell as Administrator:

```powershell
Rename-Computer -NewName "DC01" -Restart
```

## Step 3: Install AD DS and DNS

Run in PowerShell as Administrator:

```powershell
Install-WindowsFeature AD-Domain-Services,DNS -IncludeManagementTools
```

## Step 4: Promote to Domain Controller

Run in PowerShell as Administrator:

```powershell
Install-ADDSForest `
-DomainName "soc.lab" `
-DomainNetbiosName "SOC" `
-InstallDNS `
-Force
```

The server will restart.

## Step 5: Validate AD and DNS

Run after reboot:

```powershell
Get-ADDomain
Get-ADForest
ipconfig /all
nslookup dc01.soc.lab
```

## Step 6: Create OU Structure

Recommended OUs:

```text
SOC Lab
├── Admins
├── Users
├── Workstations
├── Servers
└── Service Accounts
```

Keep admin accounts separate from daily user accounts.

## Step 7: Configure NTP

Run on DC01 in PowerShell as Administrator:

```powershell
w32tm /config /manualpeerlist:"time.windows.com,0x8" /syncfromflags:manual /reliable:yes /update
Restart-Service w32time
w32tm /resync
w32tm /query /status
```

If the lab is fully air-gapped, use DC01 as the internal reference and document that external sync is disabled.

## Step 8: Install DHCP Role

Run when AD and DNS are already stable:

```powershell
Install-WindowsFeature DHCP -IncludeManagementTools
```

Authorize DHCP in AD:

```powershell
Add-DhcpServerInDC -DnsName "dc01.soc.lab" -IPAddress 172.16.20.10
```

## Step 9: Example DHCP Scope for VLAN 10

```powershell
Add-DhcpServerv4Scope -Name "VLAN10-Clients" -StartRange 172.16.10.50 -EndRange 172.16.10.200 -SubnetMask 255.255.255.0
Set-DhcpServerv4OptionValue -ScopeId 172.16.10.0 -Router 172.16.10.1 -DnsServer 172.16.20.10 -DnsDomain "soc.lab"
```

Create additional scopes for VLANs 30, 40, and 99 only if DHCP is required there.

## Step 10: Configure DHCP Relay on Palo Alto

On Palo Alto GUI:

```text
Network > DHCP > DHCP Relay
```

For each client VLAN subinterface, relay to:

```text
172.16.20.10
```

## Step 11: Join Windows Client to Domain

Client IP settings before DHCP:

| Setting | Value |
|---|---|
| IP | 172.16.10.x |
| Gateway | 172.16.10.1 |
| DNS | 172.16.20.10 |

On client PowerShell as Administrator:

```powershell
Add-Computer -DomainName soc.lab -Restart
```

## Validation

On client:

```cmd
nslookup dc01.soc.lab
ping dc01.soc.lab
whoami
```

On DC:

```powershell
Get-ADComputer -Filter *
Get-DhcpServerv4Lease -ScopeId 172.16.10.0
```
