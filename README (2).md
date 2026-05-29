# 🛡️ Snort + Wazuh Lab Environment (2-Machine Setup)

Integrating **Snort IDS** with **Wazuh SIEM** using only two machines — a Wazuh Manager and a Linux Agent that also runs Snort locally.

---

## 📐 Lab Architecture

```
┌─────────────────────────┐          ┌──────────────────────────────┐
│     Machine 1           │          │         Machine 2            │
│   Wazuh Manager         │◄────────►│   Linux Client (Wazuh Agent) │
│                         │  port    │        + Snort IDS           │
│  - Wazuh Manager        │  1514    │                              │
│  - Wazuh Indexer        │          │  - Wazuh Agent               │
│  - Wazuh Dashboard      │          │  - Snort (local monitoring)  │
└─────────────────────────┘          └──────────────────────────────┘
     192.168.1.10                            192.168.1.20
```

### Role Summary

| Machine | OS | Role |
|---|---|---|
| Machine 1 | Ubuntu 22.04 LTS | Wazuh Manager + Indexer + Dashboard |
| Machine 2 | Ubuntu 22.04 LTS | Wazuh Agent + Snort IDS |

> Snort runs on the **agent machine**, monitors its local traffic, writes alerts to a log file, and the Wazuh agent ships those alerts to the manager.

---

## ✅ Prerequisites

- Two Linux machines (physical, VM, or cloud) on the same network
- Root / sudo access on both
- Internet access for package installation
- Minimum specs:
  - Machine 1 (Manager): 4 vCPU, 8 GB RAM, 50 GB disk
  - Machine 2 (Agent): 2 vCPU, 2 GB RAM, 20 GB disk

---

## 🖥️ Machine 1 — Wazuh Manager Setup

### Install Wazuh (All-in-One)

```bash
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
sudo bash wazuh-install.sh -a
```

This installs the Wazuh manager, indexer, and dashboard in one shot.

After installation, note the admin credentials printed on screen (or retrieve them):

```bash
sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
```

Access the dashboard: `https://192.168.1.10` (admin / your-password)

### Verify Manager is Running

```bash
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard
```

### Open Required Firewall Ports

```bash
sudo ufw allow 1514/tcp    # Agent communication
sudo ufw allow 1515/tcp    # Agent registration
sudo ufw allow 443/tcp     # Dashboard
sudo ufw enable
```

---

## 🐧 Machine 2 — Wazuh Agent + Snort Setup

### Step 1 — Install Wazuh Agent

```bash
# Add Wazuh repository
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH \
  | sudo gpg --dearmor -o /usr/share/keyrings/wazuh.gpg

echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] \
  https://packages.wazuh.com/4.x/apt/ stable main" \
  | sudo tee /etc/apt/sources.list.d/wazuh.list

sudo apt update && sudo apt install -y wazuh-agent
```

Register and point the agent at the manager:

```bash
sudo WAZUH_MANAGER='192.168.1.10' \
     WAZUH_AGENT_NAME='linux-client' \
     dpkg-reconfigure wazuh-agent
```

Start the agent:

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
sudo systemctl status wazuh-agent
```

Confirm the agent appears on **Machine 1**:

```bash
sudo /var/ossec/bin/agent_control -l
```

---

### Step 2 — Install Snort on Machine 2

```bash
sudo apt update && sudo apt install -y snort
```

During install, enter:
- **Network interface** to monitor (e.g., `eth0` — check yours with `ip a`)
- **Home network** (e.g., `192.168.1.0/24`)

---

### Step 3 — Configure Snort on Machine 2

```bash
sudo nano /etc/snort/snort.conf
```

Key settings to verify/set:

```
# Your local network
ipvar HOME_NET 192.168.1.0/24
ipvar EXTERNAL_NET !$HOME_NET

# Rule path
var RULE_PATH /etc/snort/rules
include $RULE_PATH/local.rules

# Output alerts in fast format (Wazuh reads this)
output alert_fast: /var/log/snort/alert
```

Create the log directory if it doesn't exist:

```bash
sudo mkdir -p /var/log/snort
sudo chown snort:snort /var/log/snort
```

Test the config:

```bash
sudo snort -T -c /etc/snort/snort.conf -i eth0
```

Start Snort:

```bash
# Run as daemon in IDS mode
sudo snort -A fast -c /etc/snort/snort.conf -i eth0 -D -u snort -g snort

# Or enable systemd service
sudo systemctl enable snort
sudo systemctl start snort
```

---

### Step 4 — Add Custom Snort Rules on Machine 2

```bash
sudo nano /etc/snort/rules/local.rules
```

```
# ICMP ping detection
alert icmp any any -> $HOME_NET any (msg:"ICMP Ping Detected"; sid:1000001; rev:1;)

# Nmap SYN scan
alert tcp any any -> $HOME_NET any (msg:"Possible Nmap SYN Scan"; flags:S; \
  threshold: type threshold, track by_src, count 20, seconds 2; sid:1000002; rev:1;)

# SSH brute force
alert tcp any any -> $HOME_NET 22 (msg:"SSH Brute Force Attempt"; \
  threshold: type threshold, track by_src, count 5, seconds 60; sid:1000003; rev:1;)

# HTTP scan
alert tcp any any -> $HOME_NET 80 (msg:"HTTP Scan Detected"; flags:S; \
  threshold: type threshold, track by_src, count 10, seconds 5; sid:1000004; rev:1;)
```

Reload rules without restarting Snort:

```bash
sudo kill -SIGHUP $(cat /var/run/snort/snort.pid)
```

---

## 🔗 Wazuh Agent ↔ Snort Integration (Machine 2)

Tell the Wazuh agent to monitor Snort's alert log.

Edit the agent config on **Machine 2**:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add inside `<ossec_config>`:

```xml
<!-- Snort alert log monitoring -->
<localfile>
  <log_format>snort-fast</log_format>
  <location>/var/log/snort/alert</location>
</localfile>
```

Restart the agent to apply:

```bash
sudo systemctl restart wazuh-agent
```

---

## 📋 Custom Wazuh Rules (Machine 1)

On **Machine 1**, add rules to trigger on Snort alerts:

```bash
sudo nano /var/ossec/etc/rules/local_rules.xml
```

```xml
<group name="snort,ids,">

  <rule id="100001" level="8">
    <if_group>snort</if_group>
    <match>ICMP Ping Detected</match>
    <description>Snort: ICMP ping detected from $(srcip)</description>
    <group>network_scan,</group>
  </rule>

  <rule id="100002" level="10">
    <if_group>snort</if_group>
    <match>Possible Nmap SYN Scan</match>
    <description>Snort: Nmap SYN scan detected from $(srcip)</description>
    <group>network_scan,recon,</group>
  </rule>

  <rule id="100003" level="12">
    <if_group>snort</if_group>
    <match>SSH Brute Force</match>
    <description>Snort: SSH brute force attempt from $(srcip)</description>
    <group>authentication_failed,brute_force,</group>
  </rule>

</group>
```

Restart the manager to load rules:

```bash
sudo systemctl restart wazuh-manager
```

---

## 🧪 Testing the Lab

Run these commands on **Machine 2** to generate test alerts:

```bash
# Trigger ICMP rule — ping the manager
ping -c 10 192.168.1.10

# Trigger Nmap scan rule (install nmap first)
sudo apt install -y nmap
nmap -sS 192.168.1.10

# Trigger SSH brute force rule (install hydra first)
sudo apt install -y hydra
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://192.168.1.10 -t 4
```

### Verify Snort is Alerting (Machine 2)

```bash
tail -f /var/log/snort/alert
```

### Verify Wazuh is Receiving Alerts (Machine 1)

```bash
tail -f /var/ossec/logs/alerts/alerts.log | grep -i snort
```

Check the **Wazuh Dashboard** → Security Events → filter by `rule.groups: snort`

---

## 📁 Key File Reference

| File | Machine | Purpose |
|---|---|---|
| `/etc/snort/snort.conf` | Machine 2 | Snort main config |
| `/etc/snort/rules/local.rules` | Machine 2 | Custom Snort rules |
| `/var/log/snort/alert` | Machine 2 | Snort alert output |
| `/var/ossec/etc/ossec.conf` | Machine 2 | Wazuh agent config (localfile block) |
| `/var/ossec/etc/ossec.conf` | Machine 1 | Wazuh manager config |
| `/var/ossec/etc/rules/local_rules.xml` | Machine 1 | Custom Wazuh rules |
| `/var/ossec/logs/alerts/alerts.log` | Machine 1 | Wazuh alert log |

---

## 🛠️ Troubleshooting

**Agent not connecting to manager:**
```bash
# On Machine 2 — check manager IP in config
sudo grep -A5 "<client>" /var/ossec/etc/ossec.conf

# Re-register the agent
sudo /var/ossec/bin/agent-auth -m 192.168.1.10

sudo systemctl restart wazuh-agent
```

**Snort not starting:**
```bash
# Validate config
sudo snort -T -c /etc/snort/snort.conf

# Check logs
journalctl -u snort -n 50
```

**Snort alerts not appearing in Wazuh:**
```bash
# On Machine 2 — confirm the log file exists and is being written to
ls -lh /var/log/snort/alert
tail /var/log/snort/alert

# Check Wazuh agent is reading the file
sudo tail -f /var/ossec/logs/ossec.log | grep snort
```

**Firewall blocking ports:**
```bash
# On Machine 1 — allow agent traffic
sudo ufw allow 1514/tcp
sudo ufw allow 1515/tcp
sudo ufw reload
```

---

## 📚 References

- [Wazuh Documentation](https://documentation.wazuh.com)
- [Snort 3 Documentation](https://www.snort.org/documents)
- [Wazuh Log Data Collection](https://documentation.wazuh.com/current/user-manual/capabilities/log-data-collection/index.html)
- [Snort Community Rules](https://www.snort.org/downloads/#rule-downloads)

---

## 📄 License

For educational/lab use only. Do not use against systems without explicit authorization.
