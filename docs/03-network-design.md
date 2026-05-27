# Network Design

## Address Plan Summary

| Segment | Network | Purpose |
|---|---|---|
| Transit | 10.0.0.0/30 | Firewall ↔ RED router |
| VLAN 10 | 172.16.10.0/24 | Client workstations |
| VLAN 20 | 172.16.20.0/24 | Windows servers (AD, DNS, DHCP, NTP) |
| VLAN 30 | 172.16.30.0/24 | IDS management |
| VLAN 40 | 172.16.40.0/24 | SIEM (Wazuh) |
| VLAN 99 | 172.16.99.0/24 | Management |
| RED | 192.168.1.0/24 (example) | Adversary / test network |

---

## VLAN Detail

### VLAN 10 — CLIENTS

| Field | Value |
|---|---|
| Network | 172.16.10.0/24 |
| Gateway | 172.16.10.1 (Palo Alto ethernet1/2.10) |
| DHCP Server | Windows Server 2016 (VLAN 20) via firewall relay |
| DNS | 172.16.0.6 (DC01) |
| Zone | CLIENTS |
| Hosts | Domain-joined Windows workstations |

### VLAN 20 — WINDOWS-SERVERS

| Field | Value |
|---|---|
| Network | 172.16.20.0/24 |
| Gateway | 172.16.20.1 (Palo Alto ethernet1/2.20) |
| Static Hosts | DC01: 172.16.0.6 |
| DNS | 127.0.0.1 (self) / 172.16.0.6 |
| Zone | WINDOWS-SERVERS |
| Services | AD DS, DNS, DHCP, NTP |

### VLAN 30 — IDS-MGMT

| Field | Value |
|---|---|
| Network | 172.16.30.0/24 |
| Gateway | 172.16.30.1 (Palo Alto ethernet1/2.30) |
| Static Hosts | Snort sensor mgmt NIC: 172.16.30.10 |
| Zone | IDS-MGMT |
| Purpose | Snort sensor administration and log forwarding |

### VLAN 40 — SIEM

| Field | Value |
|---|---|
| Network | 172.16.40.0/24 |
| Gateway | 172.16.40.1 (Palo Alto ethernet1/2.40) |
| Static Hosts | Wazuh: 172.16.40.10 |
| Zone | SIEM |
| Purpose | Wazuh manager, log collection, syslog receiver |

### VLAN 99 — MANAGEMENT

| Field | Value |
|---|---|
| Network | 172.16.99.0/24 |
| Gateway | 172.16.99.1 (Palo Alto ethernet1/2.99) |
| Zone | MGMT |
| Purpose | Administrative access to firewall, switch, servers |

---

## Firewall Interface and Subinterface Map

```
ethernet1/1        →  10.0.0.1/30       →  RED-TRANSIT zone
ethernet1/2        →  802.1Q trunk (no native IP)
  ethernet1/2.10   →  172.16.10.1/24    →  CLIENTS zone
  ethernet1/2.20   →  172.16.20.1/24    →  WINDOWS-SERVERS zone
  ethernet1/2.30   →  172.16.30.1/24    →  IDS-MGMT zone
  ethernet1/2.40   →  172.16.40.1/24    →  SIEM zone
  ethernet1/2.99   →  172.16.99.1/24    →  MGMT zone
```

---

## Cisco Catalyst 3560 VLAN Configuration

### VLAN Database

```
vlan 10
 name CLIENTS
vlan 20
 name WINDOWS-SERVERS
vlan 30
 name IDS-MGMT
vlan 40
 name SIEM
vlan 99
 name MANAGEMENT
```

### Trunk Port to Palo Alto

```
interface GigabitEthernetX/X
 description TRUNK-TO-PALO-ALTO
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40,99
```

### Example Access Ports

```
interface GigabitEthernetX/X
 description CLIENT-WORKSTATION-01
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast

interface GigabitEthernetX/X
 description DC01-WINDOWS-SERVER
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast

interface GigabitEthernetX/X
 description SNORT-MGMT-NIC
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast

interface GigabitEthernetX/X
 description WAZUH-SIEM
 switchport mode access
 switchport access vlan 40
 spanning-tree portfast
```

### SPAN Configuration

```
monitor session 1 source interface GigabitEthernetX/X
monitor session 1 destination interface GigabitEthernetX/X
```

Source: the uplink to the Palo Alto (to capture inter-VLAN traffic) or a specific access port.  
Destination: the port connected to the Snort sniffing NIC.

---

## DHCP Relay

DHCP clients in VLAN 10 (and other VLANs) broadcast DHCP Discover on their local segment. The Palo Alto subinterface for that VLAN relays the request to the Windows Server DHCP service.

Relay configuration on Palo Alto (in Network → Virtual Router → DHCP Relay or via the interface helper-address depending on PAN-OS version):

- Relay enabled on each subinterface that needs DHCP
- Server IP: 172.16.0.6 (DC01)

Windows Server DHCP scopes:

| Scope | Network | Range | Gateway | DNS |
|---|---|---|---|---|
| CLIENTS | 172.16.10.0/24 | .100–.200 | 172.16.10.1 | 172.16.0.6 |

---

## DNS Design

All internal DNS is handled by DC01 at 172.16.0.6.

- Primary zone: `soc.lab`
- Domain controller A record: `dc01.soc.lab → 172.16.0.6`
- Clients configured via DHCP option 6 (DNS Server)
- Forward lookups: DC01 resolves internal names
- External resolution: DC01 forwards to configured forwarder (temporarily during patch windows)

---

## NTP Design

DC01 acts as the NTP server for the domain.

- DC01 syncs to an external source (temporarily during setup)
- Domain members sync to DC01 via Windows Time Service
- Non-domain devices (Snort, Wazuh) should also point to DC01 or a consistent source

Consistent time is critical for log correlation. All devices must share a common time reference.

---

## Routing Summary

| Destination | Next Hop | Method |
|---|---|---|
| 172.16.10.0/24 | directly connected (ethernet1/2.10) | Connected |
| 172.16.20.0/24 | directly connected (ethernet1/2.20) | Connected |
| 172.16.30.0/24 | directly connected (ethernet1/2.30) | Connected |
| 172.16.40.0/24 | directly connected (ethernet1/2.40) | Connected |
| 172.16.99.0/24 | directly connected (ethernet1/2.99) | Connected |
| 10.0.0.0/30 | directly connected (ethernet1/1) | Connected |
| 192.168.1.0/24 | 10.0.0.2 | OSPF |
