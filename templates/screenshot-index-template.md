# Screenshot Index

Track all evidence screenshots captured for this project. Each entry should reference the actual filename stored in `evidence/screenshots/`.

All screenshots must be sanitized before upload. Remove: passwords, license keys, serial numbers, public IPs, personal names, private emails, tokens.

---

## Index

| # | Filename | Phase | Description | Date Captured | Status |
|---|---|---|---|---|---|
| 01 | rack-front.jpg | Phase 1 | Physical rack — front view | | Pending |
| 02 | rack-cabling.jpg | Phase 1 | Cable management | | Pending |
| 03 | device-inventory.jpg | Phase 1 | Device labels and inventory | | Pending |
| 04 | pa-interface-summary.png | Phase 2 | Palo Alto `show interface all` output | | Pending |
| 05 | pa-traffic-log.png | Phase 2 | Palo Alto Traffic Monitor — inter-VLAN traffic | | Pending |
| 06 | cisco-vlan-brief.png | Phase 2 | `show vlan brief` output | | Pending |
| 07 | cisco-trunk-status.png | Phase 2 | `show interfaces trunk` output | | Pending |
| 08 | cisco-span-session.png | Phase 2 | `show monitor session 1` output | | Pending |
| 09 | ospf-neighbors.png | Phase 2 | Palo Alto OSPF neighbor state | | Pending |
| 10 | ad-ou-structure.png | Phase 3 | Active Directory — OU layout in ADUC | | Pending |
| 11 | dns-validation.png | Phase 3 | `nslookup dc01.soc.lab` output | | Pending |
| 12 | domain-join-confirmation.png | Phase 3 | Client domain membership properties | | Pending |
| 13 | ntp-sync-status.png | Phase 3 | `w32tm /query /status` output | | Pending |
| 14 | dhcp-scope.png | Phase 4 | DHCP server — scope configuration | | Pending |
| 15 | dhcp-lease-list.png | Phase 4 | Active DHCP leases | | Pending |
| 16 | snort-no-ip-on-sniff.png | Phase 5 | `ip addr` — no IP on sniffing NIC | | Pending |
| 17 | snort-alert-console.png | Phase 5 | Snort console alert output | | Pending |
| 18 | snort-alert-log.png | Phase 5 | `/var/log/snort/alert` tail | | Pending |
| 19 | wazuh-agent-list.png | Phase 6 | Wazuh — enrolled agents list | | Pending |
| 20 | wazuh-alerts-dashboard.png | Phase 6 | Wazuh — security alerts dashboard | | Pending |
| 21 | wazuh-syslog-firewall.png | Phase 6 | Wazuh — Palo Alto syslog events visible | | Pending |
| 22 | wazuh-snort-alert.png | Phase 6 | Wazuh — Snort alert ingested | | Pending |
| 23 | wazuh-timeline.png | Phase 9 | Wazuh — investigation timeline for a scenario | | Pending |

---

## Sanitization Checklist

Before uploading any screenshot, verify:

- [ ] No passwords visible
- [ ] No license keys visible
- [ ] No serial numbers visible
- [ ] No public or external IP addresses
- [ ] No personal names or usernames tied to real people
- [ ] No real email addresses
- [ ] No tokens or API keys
- [ ] No private keys or certificate content
- [ ] Full firewall backup content not included
- [ ] Packet captures with sensitive payload not included
