# Runbook: Wazuh SIEM Deployment

## Goal

Deploy Wazuh as the central SIEM for Windows events, firewall syslog, Snort alerts, and Linux host logs.

## Host Plan

| Setting | Value |
|---|---|
| Hostname | wazuh01 |
| VLAN | 40 |
| IP | 172.16.40.10 |
| Gateway | 172.16.40.1 |
| DNS | 172.16.0.6 |

## Step 1: Configure Ubuntu Host

Run on the Wazuh Ubuntu server:

```bash
sudo hostnamectl set-hostname wazuh01
sudo apt update && sudo apt upgrade -y
```

Configure static IP using Netplan according to the adapter name.

Example path:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Example config:

```yaml
network:
  version: 2
  ethernets:
    ens33:
      addresses:
        - 172.16.40.10/24
      routes:
        - to: default
          via: 172.16.40.1
      nameservers:
        addresses:
          - 172.16.0.6
```

Apply:

```bash
sudo netplan apply
ip addr
ip route
```

## Step 2: Install Wazuh All-in-One

Use the official Wazuh installation method for the version you are deploying. Example assisted install pattern:

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash wazuh-install.sh -a
```

Save generated credentials securely. Do not upload them to GitHub.

## Step 3: Validate Wazuh Services

Run on Wazuh server:

```bash
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-dashboard
sudo systemctl status wazuh-indexer
```

## Step 4: Access Dashboard

From an authorized management workstation:

```text
https://172.16.40.10
```

## Step 5: Deploy Windows Agent

From Wazuh dashboard:

```text
Agents management > Deploy new agent
```

Select:

- Windows
- Manager IP: 172.16.40.10

On Windows endpoint, validate service:

```powershell
Get-Service Wazuh
```

## Step 6: Enable Palo Alto Syslog Ingestion

Edit Wazuh manager config:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add a syslog listener example:

```xml
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>tcp</protocol>
  <allowed-ips>172.16.0.0/16</allowed-ips>
  <local_ip>172.16.40.10</local_ip>
</remote>
```

Restart manager:

```bash
sudo systemctl restart wazuh-manager
```

On Palo Alto:

```text
Device > Server Profiles > Syslog
Objects > Log Forwarding
Policies > Security > Apply Log Forwarding Profile
```

## Step 7: Validate Logs

On Wazuh server:

```bash
sudo tail -f /var/ossec/logs/archives/archives.log
sudo tail -f /var/ossec/logs/alerts/alerts.log
```

In dashboard, search for:

- Agent connection events
- Windows security events
- Palo Alto syslog messages
- Snort alert messages, once configured

## Evidence to Capture

- Dashboard login screen after successful deployment, redacted if needed
- Agent inventory
- DC01 agent connected
- Client agent connected
- Palo Alto syslog event visible
- Snort alert visible
