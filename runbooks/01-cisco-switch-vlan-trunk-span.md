# Runbook: Cisco Catalyst VLANs, Trunking, and SPAN

## Goal

Configure the Cisco Catalyst 3560 as a Layer 2 switch for VLAN access ports, trunking to the Palo Alto firewall, and SPAN mirroring for Snort IDS.

## Safety Note

Run these commands only on the lab switch. Confirm interface numbers before applying configuration.

## VLAN Creation

Run from Cisco IOS privileged EXEC mode:

```bash
enable
configure terminal

vlan 10
 name CLIENTS
vlan 20
 name WINDOWS_SERVERS
vlan 30
 name IDS_MGMT
vlan 40
 name SIEM
vlan 99
 name MGMT
end
write memory
```

## Trunk to Palo Alto Firewall

Assumption: `fa0/1` connects to Palo Alto `ethernet1/2`.

```bash
enable
configure terminal
interface fa0/1
 description TRUNK_TO_PA500_E1_2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40,99
 no shutdown
end
write memory
```

If the switch does not accept `switchport trunk encapsulation dot1q`, omit it. Some Catalyst models default to dot1q.

## Access Ports

Adjust port numbers to match the real rack.

```bash
enable
configure terminal

interface range fa0/2 - 10
 description CLIENT_PORTS
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast

interface range fa0/11 - 14
 description WINDOWS_SERVERS
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast

interface fa0/15
 description SNORT_MGMT
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast

interface fa0/16
 description WAZUH_SIEM
 switchport mode access
 switchport access vlan 40
 spanning-tree portfast

interface range fa0/17 - 18
 description MGMT_PORTS
 switchport mode access
 switchport access vlan 99
 spanning-tree portfast

end
write memory
```

## SPAN Configuration for Snort

Assumption:

- Source trunk: `fa0/1`
- Destination Snort sniffing NIC: `fa0/24`

```bash
enable
configure terminal
monitor session 1 source interface fa0/1 both
monitor session 1 destination interface fa0/24
end
write memory
```

## Validation Commands

```bash
show vlan brief
show interfaces trunk
show monitor session 1
show running-config interface fa0/1
show running-config interface fa0/24
```

## Expected Results

- VLANs 10, 20, 30, 40, and 99 exist.
- Firewall uplink is trunking required VLANs.
- SPAN session mirrors traffic to the Snort sniffing port.
- Snort destination port is not used as a normal access port.
