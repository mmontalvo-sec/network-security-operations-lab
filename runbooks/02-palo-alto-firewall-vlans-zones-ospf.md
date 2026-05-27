# Runbook: Palo Alto PA-500 VLAN Subinterfaces, Zones, and OSPF

## Goal

Configure the Palo Alto PA-500 as the Layer 3 gateway, firewall policy point, and OSPF peer for the segmented SOC lab.

## Interface Plan

| Interface | Role | Address |
|---|---|---:|
| ethernet1/1 | Transit to RED router | 10.0.0.1/30 |
| ethernet1/2 | Trunk to Cisco switch | No untagged IP |
| ethernet1/2.10 | VLAN 10 Clients | 172.16.10.1/24 |
| ethernet1/2.20 | VLAN 20 Windows Servers | 172.16.20.1/24 |
| ethernet1/2.30 | VLAN 30 IDS Management | 172.16.30.1/24 |
| ethernet1/2.40 | VLAN 40 SIEM | 172.16.40.1/24 |
| ethernet1/2.99 | VLAN 99 Management | 172.16.99.1/24 |

## GUI Steps

### 1. Create Zones

Go to:

```text
Network > Zones
```

Create:

- CLIENTS
- WINDOWS-SERVERS
- IDS-MGMT
- SIEM
- MGMT
- RED-TRANSIT

### 2. Create Virtual Router

Go to:

```text
Network > Virtual Routers
```

Create:

```text
VR-SOC
```

Add the following interfaces:

- ethernet1/1
- ethernet1/2.10
- ethernet1/2.20
- ethernet1/2.30
- ethernet1/2.40
- ethernet1/2.99

### 3. Configure Transit Interface

Go to:

```text
Network > Interfaces > Ethernet > ethernet1/1
```

Set:

- Type: Layer3
- Virtual Router: VR-SOC
- Security Zone: RED-TRANSIT
- IPv4: 10.0.0.1/30

### 4. Configure VLAN Subinterfaces

Go to:

```text
Network > Interfaces > Ethernet > ethernet1/2 > Add Subinterface
```

Create:

| Subinterface | Tag | Zone | IP |
|---|---:|---|---:|
| ethernet1/2.10 | 10 | CLIENTS | 172.16.10.1/24 |
| ethernet1/2.20 | 20 | WINDOWS-SERVERS | 172.16.20.1/24 |
| ethernet1/2.30 | 30 | IDS-MGMT | 172.16.30.1/24 |
| ethernet1/2.40 | 40 | SIEM | 172.16.40.1/24 |
| ethernet1/2.99 | 99 | MGMT | 172.16.99.1/24 |

Each subinterface must be Layer3, assigned to `VR-SOC`, and placed in the correct security zone.

### 5. Configure Initial Security Policies

Go to:

```text
Policies > Security
```

Create temporary validation rules:

| Rule | Source Zone | Destination Zone | Action | Logging |
|---|---|---|---|---|
| Clients to Servers | CLIENTS | WINDOWS-SERVERS | Allow | Session end |
| Servers to Clients | WINDOWS-SERVERS | CLIENTS | Allow | Session end |
| Internal to SIEM | CLIENTS/WINDOWS-SERVERS/IDS-MGMT/MGMT | SIEM | Allow | Session end |
| Management Access | MGMT | Internal zones | Allow as needed | Session end |
| Default Deny | Any | Any | Deny | Session end |

Replace broad rules later with specific DNS, Kerberos, LDAP, SMB, WinRM/RDP, syslog, and agent traffic rules.

### 6. Configure OSPF

Go to:

```text
Network > Virtual Routers > VR-SOC > OSPF
```

Set:

- Router ID: 10.255.255.1
- Area: 0.0.0.0
- OSPF interface: ethernet1/1

Advertise internal VLAN routes according to your PAN-OS version and export policy model.

## RED Router Example

On the Cisco RED router:

```bash
enable
configure terminal
router ospf 1
 network 10.0.0.0 0.0.0.3 area 0
 network 192.168.1.0 0.0.0.255 area 0
end
write memory
```

## Validation

On Palo Alto CLI:

```bash
show interface all
show routing route
```

On Cisco RED router:

```bash
show ip ospf neighbor
show ip route
```

Expected:

- Palo Alto subinterfaces are up.
- Connected routes for all VLANs exist.
- OSPF adjacency forms with RED router.
- RED router learns SOC VLAN routes.
- Firewall traffic logs show inter-zone traffic.
