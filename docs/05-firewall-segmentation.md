# Firewall Segmentation — Palo Alto PA-500

## Design Decision

The Palo Alto PA-500 is the Layer 3 gateway for all internal VLANs. Every inter-VLAN
flow crosses the firewall — making it inspectable, enforceable, and loggable.

The Cisco Catalyst 3560 stays Layer 2 only. No SVIs. No routed ports for user traffic.
If the switch did inter-VLAN routing, the firewall would never see that traffic.
That defeats the entire purpose of the architecture.

---

## Physical Interface Layout

| Interface | IP / Config | Role |
|---|---|---|
| ethernet1/1 | 10.10.10.1/30 | Transit to RED router — RED-TRANSIT zone |
| ethernet1/2 | Trunk, no IP | 802.1Q trunk to BlueSwitch |
| ethernet1/2.10 | 172.16.10.1/24 | CLIENTS zone gateway |
| ethernet1/2.20 | 172.16.0.1/24 | WINDOWS-SERVERS zone gateway |
| ethernet1/2.30 | 172.16.30.1/24 | IDS-MGMT zone gateway |
| ethernet1/2.40 | 172.16.40.1/24 | SIEM zone gateway |
| ethernet1/2.99 | 172.16.99.1/24 | MGMT zone gateway |
| ethernet1/3 | DHCP (temporary) | Internet access during setup only |

Virtual Router: **VR-SOC** — all interfaces assigned here.

---

## Security Zones

| Zone | Interface | Trust Level | Purpose |
|---|---|---|---|
| RED-TRANSIT | ethernet1/1 | Untrusted | RED network / adversary segment |
| CLIENTS | ethernet1/2.10 | Low | Domain-joined workstations |
| WINDOWS-SERVERS | ethernet1/2.20 | Medium | AD DS, DNS, DHCP, file share |
| IDS-MGMT | ethernet1/2.30 | High | Snort sensor management |
| SIEM | ethernet1/2.40 | High | Wazuh SIEM platform |
| MGMT | ethernet1/2.99 | Highest | Administrative access |

---

## Subinterface Configuration (PAN-OS CLI)

```bash
# Set parent interface as trunk (no IP)
set network interface ethernet ethernet1/2 layer3 units

# VLAN 10 — CLIENTS
set network interface ethernet ethernet1/2 layer3 units ethernet1/2.10 tag 10
set network interface ethernet ethernet1/2 layer3 units ethernet1/2.10 ip 172.16.10.1/24
set network interface ethernet ethernet1/2 layer3 units ethernet1/2.10 zone CLIENTS

# VLAN 20 — WINDOWS-SERVERS
set network interface ethernet ethernet1/2 layer3 units ethernet1/2.20 tag 20
set network interface ethernet ethernet1/2 layer3 units ethernet1/2.20 ip 172.16.0.1/24
set network interface ethernet ethernet1/2 layer3 units ethernet1/2.20 zone WINDOWS-SERVERS

# VLAN 30 — IDS-MGMT
set network interface ethernet ethernet1/2 layer3 units ethernet1/2.30 tag 30
set network interface ethernet ethernet1/2 layer3 units ethernet1/2.30 ip 172.16.30.1/24
set network interface ethernet ethernet1/2 layer3 units ethernet1/2.30 zone IDS-MGMT

# VLAN 40 — SIEM
set network interface ethernet ethernet1/2 layer3 units ethernet1/2.40 tag 40
set network interface ethernet ethernet1/2 layer3 units ethernet1/2.40 ip 172.16.40.1/24
set network interface ethernet ethernet1/2 layer3 units ethernet1/2.40 zone SIEM

# VLAN 99 — MGMT
set network interface ethernet ethernet1/2 layer3 units ethernet1/2.99 tag 99
set network interface ethernet ethernet1/2 layer3 units ethernet1/2.99 ip 172.16.99.1/24
set network interface ethernet ethernet1/2 layer3 units ethernet1/2.99 zone MGMT
```

---

## OSPF Configuration

OSPF Area 0 runs between the Palo Alto and the RED router over the 10.10.10.0/30 transit.

```bash
set network virtual-router VR-SOC protocol ospf enable yes
set network virtual-router VR-SOC protocol ospf router-id 10.10.10.1
set network virtual-router VR-SOC protocol ospf area 0.0.0.0 interface ethernet1/1 enable yes
```

Validated: RED router OSPF router-ID 1.1.1.1, network statements confirmed in screenshot.

---

## Security Policy (Initial — Any/Any for Baseline)

During baseline collection, an initial permissive rule was used to allow all inter-zone
traffic. This is intentional — the goal in Phase 1 is to validate connectivity and collect
baseline logs, not enforce least-privilege yet.

```
Rule: Allow-All-Interzone
Source zone:      any
Destination zone: any
Application:      any
Action:           allow
Log:              yes (log at session end)
```

Phase 7 (planned) replaces this with least-privilege zone-pair rules.

---

## Policy Hardening Plan (Phase 7)

| Source Zone | Destination Zone | Required Ports | Application |
|---|---|---|---|
| CLIENTS | WINDOWS-SERVERS | 53, 88, 389, 445, 3389 | DNS, Kerberos, LDAP, SMB, RDP |
| CLIENTS | SIEM | 1514, 1515 | Wazuh agent |
| WINDOWS-SERVERS | SIEM | 1514, 1515 | Wazuh agent |
| IDS-MGMT | SIEM | 1514, 1515 | Wazuh agent |
| any | any | — | deny + log (default) |

---

## Validation Commands

```bash
# PAN-OS CLI
show interface all
show routing route
show routing protocol ospf neighbor
show routing protocol ospf runtime

# Check traffic logs via GUI
Monitor → Traffic → filter: (zone.src eq CLIENTS) and (zone.dst eq WINDOWS-SERVERS)
```

---

## Evidence to Capture

- [x] Palo Alto subinterface table screenshot (confirmed — all zones visible)
- [ ] Palo Alto security rulebase screenshot
- [ ] Traffic Monitor showing inter-VLAN allowed traffic
- [ ] Traffic Monitor showing denied traffic (after Phase 7)
- [ ] OSPF neighbor state screenshot
