# Snort IDS — Passive Visibility

## Design Decision

Snort is deployed as a **passive** sensor. It receives a copy of traffic via SPAN
and generates alerts without sitting inline. This means:

- Snort cannot disrupt traffic — no inline failure risk
- The sensor has zero network exposure on its sniffing interface
- Visibility does not depend on Snort staying up

The sniffing NIC has **no IP address**. This is not an oversight — it is the correct
architecture for passive IDS deployment.

---

## Confirmed SPAN Configuration (from BlueSwitch screenshot)

```
monitor session 1 source interface Fa0/1 - 2, Fa0/12 - 16
monitor session 1 destination interface Fa0/9
```

| Element | Value |
|---|---|
| Source ports | Fa0/1 (uplink to PA), Fa0/2, Fa0/12–16 (clients + SIEM) |
| Destination port | Fa0/9 (connected to Snort sniffing NIC) |
| Session | 1 |
| Switch | BlueSwitch (Cisco Catalyst 3560) |

---

## Snort Host Architecture

| NIC | Interface | IP | Purpose |
|---|---|---|---|
| Management | ens33 (example) | 172.16.30.9 | Admin, SSH, agent comms, log forwarding |
| Sniffing | ens34 (example) | None (by design) | SPAN traffic capture only |

Hardware note: a PCIe dual-NIC card was added to the Snort host specifically to support
this dual-NIC architecture.

---

## Installation

```bash
# Ubuntu Server
sudo apt update
sudo apt install -y snort

# Verify installation
snort --version
```

---

## Configuration

Edit `/etc/snort/snort.conf`:

```bash
# Set HOME_NET to all internal lab subnets
var HOME_NET [172.16.10.0/24,172.16.0.0/24,172.16.30.0/24,172.16.40.0/24,172.16.99.0/24]
var EXTERNAL_NET !$HOME_NET
```

---

## Network Interface Configuration (Netplan)

```yaml
network:
  version: 2
  ethernets:
    ens33:
      addresses:
        - 172.16.30.9/24
      routes:
        - to: default
          via: 172.16.30.1
      nameservers:
        addresses:
          - 172.16.0.6
    ens34:
      dhcp4: false
      optional: true
```

Apply: `sudo netplan apply`

---

## Validation

```bash
# Confirm no IP on sniffing interface
ip addr show ens34

# Run Snort in console mode against sniffing interface
sudo snort -i ens34 -A console -q

# Check alert log
sudo tail -f /var/log/snort/alert

# Confirm management interface is up
ip addr show ens33
ping 172.16.30.1
```

---

## Alert Forwarding to Wazuh

Install Wazuh agent on the Snort host:

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | apt-key add -
# Add repo and install wazuh-agent
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

Edit `/var/ossec/etc/ossec.conf` — add localfile block:

```xml
<localfile>
  <log_format>snort-fast</log_format>
  <location>/var/log/snort/alert</location>
</localfile>
```

```bash
sudo systemctl restart wazuh-agent
```

---

## Evidence to Capture

- [ ] `ip addr` output showing ens34 with no IP address
- [ ] Snort console alert output from a test ping sweep
- [ ] `/var/log/snort/alert` content
- [ ] Wazuh dashboard showing Snort alert ingested
- [ ] `show monitor session 1` on BlueSwitch (confirmed in screenshot)
