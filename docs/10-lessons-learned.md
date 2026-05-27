# Lessons Learned

Real lessons from building and operating this lab on physical hardware.

---

## Network and Architecture

**Firewall-centric inter-VLAN routing is the right call — but harder than it looks.**
Setting up the Palo Alto as the Layer 3 gateway for all VLANs instead of using the
switch in routed mode required trunk configuration, tagged subinterfaces, and correct
zone assignment. When it works, every inter-VLAN flow is loggable and enforceable.
When something is misconfigured, traffic silently drops with no obvious error.

**Trunk misconfigurations are the hardest to diagnose.**
A trunk port that is up but missing a VLAN tag causes traffic for that VLAN to fail
completely — and it looks exactly like a routing problem or a firewall rule problem.
Validate `show interfaces trunk` and `show vlan brief` before blaming anything else.

**Subinterface tag mismatch is a silent killer.**
If the Palo Alto subinterface tag does not match the VLAN ID on the switch, traffic
is dropped at the VLAN boundary with no error message. Always verify that the tag on
the subinterface matches exactly what the switch expects.

**OSPF convergence requires clean /30 addressing and matching areas.**
Misconfigured wildcard masks in the OSPF `network` statement caused incomplete
routing advertisements. The `show ip ospf neighbor` command confirmed when neighbors
were missing. Once the wildcard mask matched the /30 correctly, routes converged.

---

## Active Directory and Windows

**Static IP and DNS must be correct before AD DS installation.**
If the server has the wrong IP or DNS pointing to an external server at promotion time,
AD DS will fail or install with configuration that breaks later. Set the static IP,
verify gateway reachability, then promote.

**Time synchronization is not optional for Wazuh.**
Wazuh agent authentication fails with a time skew greater than ~5 minutes. This is not
obvious from the error message. The fix is `w32tm /resync /force` on the DC, and
pointing all Linux hosts to the same NTP source before enrolling agents.

**Domain join requires DNS to work first.**
If the Windows client cannot resolve `soc.lab` before joining, the join fails. The most
common cause is the firewall blocking DNS (port 53) between CLIENTS and WINDOWS-SERVERS.
Check the Palo Alto traffic log before troubleshooting the client.

**Separating DHCP from the DC is correct but adds relay complexity.**
Having a dedicated DHCP server on a different host from the DC is more realistic, but it
requires configuring DHCP relay on the Palo Alto subinterfaces. A misconfigured relay
causes clients to not receive leases and is hard to trace because DHCP broadcasts are
silent from the firewall's perspective.

---

## SIEM and IDS

**Baseline volume matters more than alert count.**
At 20,887 events before any adversarial activity, almost every alert rule fires on
normal behavior. Without a baseline, every event looks significant. With a baseline,
the 8 authentication failures during the demo were immediately visible because they
were anomalous against 233 normal successes.

**SPAN configuration must be verified independently.**
After configuring `monitor session 1` on the switch, Snort appeared to be receiving
traffic but was actually silent. The sniffing interface must be confirmed receiving
frames using `tcpdump -i <sniff_int> -n` before trusting Snort is seeing anything.

**Wazuh agent disconnection is a detectable event — and should be treated as one.**
When a monitored host goes offline unexpectedly, Wazuh generates a disconnect alert.
In a real SOC this is meaningful — an agent that stops reporting could mean the host
is down, the agent was removed, or the network path broke. This was validated during
the lab demonstration.

---

## Physical Lab

**Real hardware surfaces problems that virtual labs never show.**
A switch failed mid-lab. A PCIe NIC card had to be installed for the dual-NIC IDS
design. A firewall factory reset was required after a misconfiguration. Fan warnings
appeared in router console output. None of these happen in GNS3 or Packet Tracer.

**Cable management is documentation.**
Messy cables make physical troubleshooting significantly harder. During the lab build,
unlabeled cables caused port assignment confusion. A simple cable label or port map
saves time every time something needs to be changed.

**Factory reset is not failure — it is the correct recovery procedure.**
The Palo Alto PA-500 was factory reset mid-project after a misconfiguration made the
management interface unreachable. The recovery procedure (boot into maint mode,
factory reset, set management IP, reconnect) is a real skill. Being able to recover
a firewall from a bad state is more valuable than never having needed to.

---

## Documentation

**Document the actual state, not the intended state.**
The VLAN 20 subnet is 172.16.0.0/24 in the actual configuration — not 172.16.20.0/24
as originally designed. The documentation reflects the real configuration because a
reviewer who checks the screenshots will see the discrepancy immediately.

**Inconsistent documentation is worse than incomplete documentation.**
Having the README say one IP and the screenshot show another destroys credibility.
It is better to document what is actually configured, even if it differs from the plan.

**Write runbooks for yourself six months from now.**
The runbooks in this repo were written assuming the reader knows nothing about the
environment. That is the correct level of detail — because in six months, you will
have forgotten most of it.
