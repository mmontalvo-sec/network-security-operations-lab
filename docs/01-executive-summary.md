# Executive Summary

**Project:** Network Security Operations Lab  
**Domain:** soc.lab  
**Classification:** Portfolio / Educational — Authorized Lab Environment Only

---

## What This Project Is

This is a completed, physically-built Network Security Operations lab designed to replicate the architecture, tooling, and workflows of a real enterprise Security Operations Center (SOC).

The lab was not a follow-along exercise. It was proposed, designed, and built by the student who led the technical direction, created implementation guides, configured the infrastructure, and demonstrated the completed system. The project also served as teaching material — the design was explained to other students and the demonstration scenarios were built specifically to show how a real SOC monitors, detects, and investigates events.

---

## Why It Was Built

Defensive security skills require real environments to practice in. Reading about firewall zones, SIEM log correlation, and IDS alert validation teaches concepts — building them on physical hardware teaches how they actually fail, how they depend on each other, and what it takes to make them work together reliably.

The lab was built specifically to demonstrate competency in areas directly relevant to SOC Analyst, Network Security Analyst, and NOC Technician roles: segmentation, identity, logging, visibility, and investigation.

---

## What Was Built

**Physical layer:** A racked, cabled, multi-device environment using real enterprise hardware. Real cables, real failure modes, and real troubleshooting — including a switch failure, a firewall factory reset, and a hardware NIC card addition for the IDS sensor.

**Network segmentation:** Five VLANs enforced through a Palo Alto PA-500 firewall acting as the inter-VLAN Layer 3 gateway. The Cisco Catalyst switch (BlueSwitch) stays Layer 2 only — every inter-VLAN flow crosses the firewall for inspection and logging.

**Identity infrastructure:** Windows Server 2016 Active Directory domain with three domain-joined server hosts (primary DC/DNS, DHCP server, backup/file share server) and two domain-joined Windows 10 clients. Three AD user accounts created for demonstration scenarios.

**SIEM monitoring:** Wazuh all-in-one deployment with five active agents — two Windows 10 clients, one red team host, and two Windows Server 2016 hosts. Over 20,000 events collected including authentication success and failure events mapped to the MITRE ATT&CK framework.

**IDS visibility:** Snort deployed on an Ubuntu Server with a dual-NIC architecture. The switch mirrors traffic from upstream/firewall and client ports to the Snort sniffing NIC via SPAN. Snort receives a copy of real traffic without sitting inline.

**Detection scenarios:** Eight structured scenarios executed during lab demonstration including nmap scanning, failed SMB authentication, successful login correlation, shared folder access, and file integrity monitoring.

---

## Architecture Summary

```
RED Network (192.168.1.0/24)
    → RED Router (OSPF, 10.10.10.2/30)
        → [Transit 10.10.10.0/30]
            → Palo Alto PA-500 (inter-VLAN gateway, zone enforcement, syslog)
                → Cisco Catalyst 3560 (BlueSwitch, Layer 2 only)
                    → VLAN 10 (Clients, 172.16.10.0/24)
                    → VLAN 20 (Windows Servers, 172.16.0.x / 172.16.20.0/24)
                    → VLAN 30 (IDS Mirror, 172.16.30.0/24)
                    → VLAN 40 (SIEM, 172.16.40.0/24)
                    → VLAN 99 (Management, 172.16.99.0/24)
                    → SPAN port → Snort (passive sensor, no IP on sniffing NIC)
                        → Wazuh SIEM (all log sources centralized)
```

---

## What This Demonstrates for Employers

A candidate who built this lab understands:

- How to design a VLAN-segmented network and why segmentation matters
- How to configure a Palo Alto firewall as a security control point — not just a router
- How Active Directory integrates with the rest of the environment (DHCP, DNS, client auth)
- How Wazuh SIEM is deployed and how agents work across Windows endpoints
- How to set up passive IDS monitoring with Snort using a SPAN port
- How to correlate logs from multiple sources to build an investigation timeline
- How to write technical documentation that others can follow
- How real hardware behaves differently from simulated environments

This is not a list of tools used — it is a completed system that was built, validated, and demonstrated.

---

## Current Status

The lab is complete as a functional SOC training environment. Phases 1–6 (physical build, segmentation, Windows infrastructure, SIEM, IDS, and detection scenarios) are fully implemented and demonstrated.

Planned next steps: firewall policy hardening to least-privilege, Sysmon deployment for enhanced endpoint visibility, and formal incident report documents for each detection scenario.
