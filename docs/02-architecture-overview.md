# Architecture Overview

## Design Philosophy

The architecture is built around one central decision: **the firewall is the control plane**.

Rather than using a multilayer switch (like the Catalyst 3560 in routed mode) as the inter-VLAN gateway, all internal VLAN traffic is routed through the Palo Alto PA-500. This means:

- Every inter-VLAN flow crosses a security zone boundary
- Every inter-VLAN flow is subject to firewall policy
- Every inter-VLAN flow is logged
- Future policy hardening requires no topology changes

The cost is slightly more complex initial configuration. The benefit is a defensible, loggable, enforceable architecture that reflects how production enterprise networks should be built.

---

## Physical Topology

```
[RED Network / Adversary Segment]
        |
   [RED Router]  ← 10.0.0.2/30
        |
   [Transit: 10.0.0.0/30]  ← OSPF
        |
   [Palo Alto PA-500]  ← 10.0.0.1/30 on ethernet1/1
        |
   [802.1Q Trunk on ethernet1/2]
        |
   [Cisco Catalyst 3560 - L2 only]
        |
   ┌────┼────────┬──────────┬─────────┐
VLAN10  VLAN20  VLAN30    VLAN40   VLAN99
Clients WinSrv  IDS-Mgmt  SIEM     Mgmt
                   |
               [Snort sensor
                sniff NIC → SPAN port on switch
                mgmt NIC → VLAN30]
```

---

## Firewall Interface Layout

| Interface | Configuration | Role |
|---|---|---|
| ethernet1/1 | 10.0.0.1/30 | Transit to RED router |
| ethernet1/2 | 802.1Q trunk, no IP | Trunk to Catalyst 3560 |
| ethernet1/2.10 | 172.16.10.1/24, VLAN 10 | CLIENTS zone gateway |
| ethernet1/2.20 | 172.16.20.1/24, VLAN 20 | WINDOWS-SERVERS zone gateway |
| ethernet1/2.30 | 172.16.30.1/24, VLAN 30 | IDS-MGMT zone gateway |
| ethernet1/2.40 | 172.16.40.1/24, VLAN 40 | SIEM zone gateway |
| ethernet1/2.99 | 172.16.99.1/24, VLAN 99 | MGMT zone gateway |

---

## Security Zones

| Zone | Interfaces | Trust Level | Purpose |
|---|---|---|---|
| CLIENTS | ethernet1/2.10 | Untrusted endpoint | User workstations |
| WINDOWS-SERVERS | ethernet1/2.20 | Trusted infrastructure | AD DS, DNS, DHCP, NTP |
| IDS-MGMT | ethernet1/2.30 | Trusted monitoring | Snort management |
| SIEM | ethernet1/2.40 | Trusted monitoring | Wazuh platform |
| MGMT | ethernet1/2.99 | Highly trusted | Administrative access |
| RED-TRANSIT | ethernet1/1 | Untrusted / adversary | RED router, test traffic |

---

## OSPF Design

OSPF is used to route between the Palo Alto firewall and the RED network router.

- Area: 0 (backbone)
- Palo Alto advertises internal lab subnets (or a default) into OSPF
- RED router learns internal routes via OSPF
- This allows controlled adversary simulation from the RED segment without requiring static routes

OSPF is not used internally between VLANs — the Palo Alto handles that directly via connected subinterfaces.

---

## Snort IDS Architecture

Snort is deployed as a **passive network sensor** using a dual-NIC design:

| NIC | Interface | IP Address | Purpose |
|---|---|---|---|
| Management | ens33 (example) | 172.16.30.10/24 | Admin, agent comms, log forwarding |
| Sniffing | ens34 (example) | None (intentional) | SPAN traffic capture only |

The Cisco Catalyst 3560 mirrors a source port (or source VLAN) to the destination port connected to the Snort sniffing NIC.

**Why no IP on the sniffing NIC?**  
A sniffing interface with no IP address cannot receive or initiate connections. This reduces the sensor's network exposure and is consistent with passive monitoring best practice.

---

## Wazuh SIEM Log Sources

| Source | Log Type | Collection Method | Wazuh Processing |
|---|---|---|---|
| Windows DC | Security event log | Wazuh agent | Windows audit events (4624, 4625, 4648, 4768, etc.) |
| Windows clients | Security event log | Wazuh agent | Logon, failed auth, object access |
| Palo Alto firewall | Syslog | UDP/514 to Wazuh | Traffic, threat, URL logs |
| Snort | Alert log | Agent localfile `/var/log/snort/alert` | IDS alerts |

---

## RED Network Design

The RED network is the adversary / testing segment.

- Physically separated from SOC VLANs
- Routed into the SOC-side via OSPF over the 10.0.0.0/30 transit
- The Palo Alto firewall controls what RED-sourced traffic can reach SOC VLANs
- All testing from RED is authorized, documented, and scoped

During baseline phases, the RED network is administratively blocked by firewall policy. Access is enabled only during controlled testing windows.

---

## Internet Access

Internet access is treated as a **temporary and controlled service** in this lab.

The lab is designed to simulate an air-gapped or near-air-gapped enterprise/security lab. Internet is enabled briefly and intentionally for:
- Package and patch downloads
- Wazuh/Snort rule updates
- Windows Update

After updates, external access is disabled. This prevents the lab from becoming a persistent internet-exposed environment and simulates environments where outbound access is tightly controlled.

---

## Key Design Tradeoffs

| Decision | Alternative Considered | Reason for Choice |
|---|---|---|
| Firewall as L3 gateway | Catalyst 3560 in routed mode | Firewall provides logging and enforcement; switch does not |
| Passive Snort | Inline IDS | Avoids traffic disruption risk during lab operation |
| SIEM on dedicated VLAN | SIEM on Windows server VLAN | Architectural separation; different trust and access requirements |
| DHCP on Windows Server | DHCP on firewall | More enterprise-realistic; firewall acts as relay only |
| Air-gapped / controlled internet | Persistent outbound | Simulates real restricted lab environments; reduces attack surface |
