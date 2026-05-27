# Runbook: Snort IDS Deployment

## Goal

Deploy Snort as a passive IDS sensor using SPAN/mirrored traffic from the Cisco switch.

## Host Plan

| Interface | Purpose | IP |
|---|---|---|
| Management NIC | SSH/admin/log forwarding | 172.16.30.10/24 |
| Sniffing NIC | SPAN/mirror traffic | None |

## Critical Rule

Do not assign an IP address to the Snort sniffing NIC. If you do, congratulations, you made the sensor weird for no reason.

## Step 1: Identify Interfaces

Run on Ubuntu Snort server:

```bash
ip link
ip addr
```

Record:

- Management interface name
- Sniffing interface name

## Step 2: Configure Management IP

Edit Netplan:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Example:

```yaml
network:
  version: 2
  ethernets:
    ens33:
      addresses:
        - 172.16.30.10/24
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

Apply:

```bash
sudo netplan apply
ip addr
ip route
```

## Step 3: Install Snort

```bash
sudo apt update
sudo apt install -y snort
```

If the package differs on your Ubuntu version:

```bash
apt-cache search snort
```

## Step 4: Configure HOME_NET

For Snort 2, edit:

```bash
sudo nano /etc/snort/snort.conf
```

Set HOME_NET to include internal lab networks:

```text
var HOME_NET [172.16.10.0/24,172.16.20.0/24,172.16.30.0/24,172.16.40.0/24,172.16.99.0/24]
```

## Step 5: Test Snort on Sniffing Interface

Replace `<SNIFF_INTERFACE>` with the real interface name:

```bash
sudo snort -i <SNIFF_INTERFACE> -A console
```

Generate normal traffic from another host:

```cmd
ping 172.16.0.6
nslookup dc01.soc.lab
```

## Step 6: Configure SPAN on Switch

See `runbooks/01-cisco-switch-vlans-span.md`.

## Step 7: Generate Safe Test Alert

From an authorized lab test host only:

```bash
nmap -sS 172.16.0.6
```

Record:

- Start time
- Source IP
- Destination IP
- Expected detection

## Step 8: Forward Snort Logs to Wazuh

Install Wazuh agent on the Snort host.

Edit local agent config:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Example localfile block:

```xml
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/snort/alert</location>
</localfile>
```

Restart:

```bash
sudo systemctl restart wazuh-agent
```

## Validation

```bash
ip addr
sudo systemctl status snort
sudo tail -f /var/log/snort/alert
sudo systemctl status wazuh-agent
```

Expected:

- Management NIC has IP.
- Sniffing NIC has no IP.
- Snort sees mirrored traffic.
- Snort alert appears locally.
- Wazuh ingests alert.
