# Wazuh Lab — Exam Guide
## Syslog + Sysmon + Atomic Red Team (ATT&CK Emulation)

---

## YOUR LAB SETUP

Before starting, collect your IPs by running this on each machine:

```bash
ip a        # on Linux machines (Manager and Linux Agent)
ipconfig    # on Windows machine
```

| Machine   | Role                    | Private IP        |
|-----------|-------------------------|-------------------|
| Machine 1 | Wazuh Manager (Linux)   | e.g. 192.168.x.x  |
| Machine 2 | Linux Agent             | e.g. 192.168.x.x  |
| Machine 3 | Windows Agent           | e.g. 192.168.x.x  |

---

## OVERVIEW — What You Will Do

```
STEP 1  →  Configure Syslog on Wazuh Manager (Linux)
STEP 2  →  Install and Configure Sysmon on Windows Agent
STEP 3  →  Tell Windows Agent to Forward Sysmon Events to Manager
STEP 4  →  Install Atomic Red Team on Windows Agent
STEP 5  →  Add Detection Rules on Wazuh Manager
STEP 6  →  Run ATT&CK Emulation Tests
STEP 7  →  Verify Alerts on Wazuh Dashboard
```

---

# STEP 1 — Configure Syslog on Wazuh Manager

> **Where:** SSH into your Wazuh Manager (Linux machine)  
> **What this does:** Makes the Manager listen on port 514 to receive syslog from other devices

### 1.1 — Find your IPs

Run on **Wazuh Manager**:
```bash
ip a
```
Run on **Linux Agent**:
```bash
ip a
```
Note both `inet` addresses — you will need them below.

---

### 1.2 — Edit the Wazuh Manager config file

```bash
sudo nano /var/ossec/etc/ossec.conf
```

---

### 1.3 — Add the syslog block

Scroll to the bottom. Find `</ossec_config>` and paste this block **just BEFORE** it:

```xml
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>tcp</protocol>
  <allowed-ips>LINUX_AGENT_PRIVATE_IP/24</allowed-ips>
  <local_ip>MANAGER_PRIVATE_IP</local_ip>
</remote>
```

**Replace the placeholders:**

| Placeholder                  | Replace with                         | Example             |
|------------------------------|--------------------------------------|---------------------|
| `LINUX_AGENT_PRIVATE_IP/24`  | Private IP of your Linux Agent + /24 | `192.168.1.50/24`   |
| `MANAGER_PRIVATE_IP`         | Private IP of your Wazuh Manager     | `192.168.1.10`      |

**Real example — what your block should look like:**
```xml
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>tcp</protocol>
  <allowed-ips>192.168.1.50/24</allowed-ips>
  <local_ip>192.168.1.10</local_ip>
</remote>
```

**What each line means:**

| Tag               | Meaning                                                              |
|-------------------|----------------------------------------------------------------------|
| `<connection>`    | Accept syslog format (not agent format)                              |
| `<port>`          | Listen on port 514 (standard syslog port)                            |
| `<protocol>`      | Use TCP (reliable delivery)                                          |
| `<allowed-ips>`   | IP of the device SENDING syslog TO you (your Linux Agent). /24 = whole subnet. **Mandatory.** |
| `<local_ip>`      | The Manager's own IP address to listen on                            |

---

### 1.4 — Save the file

Press `Ctrl + X` → `Y` → `Enter`

---

### 1.5 — Restart Wazuh Manager

```bash
sudo systemctl restart wazuh-manager
```

---

### 1.6 — Verify port 514 is listening

```bash
sudo ss -tlnp | grep 514
```

Expected output:
```
LISTEN   0   128   192.168.1.10:514   0.0.0.0:*   users:(("wazuh-remoted",...))
```

✅ If you see `wazuh-remoted` listening on 514 — syslog config is working.

---

### 1.7 — Test: Send a syslog message from Linux Agent

On **Linux Agent**:
```bash
logger -n MANAGER_PRIVATE_IP -P 514 --tcp "This is a test syslog message from Linux Agent"
```

On **Wazuh Manager**, confirm it arrived:
```bash
sudo tail -f /var/ossec/logs/archives/archives.log
```
You should see the test message. Press `Ctrl+C` to stop.

---

# STEP 2 — Install and Configure Sysmon on Windows Agent

> **Where:** Windows machine (Windows Agent) — PowerShell as Administrator  
> **What this does:** Installs Sysmon so Windows logs detailed security events that Wazuh can read

### 2.1 — Open PowerShell as Administrator

`Windows key` → type `PowerShell` → Right-click → **Run as administrator** → **Yes**

---

### 2.2 — Create a working folder

```powershell
mkdir C:\Tools
cd C:\Tools
```

---

### 2.3 — Download Sysmon

```powershell
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" -OutFile "C:\Tools\Sysmon.zip"
```

---

### 2.4 — Extract Sysmon

```powershell
Expand-Archive -Path "C:\Tools\Sysmon.zip" -DestinationPath "C:\Tools\Sysmon" -Force
```

---

### 2.5 — Download the Sysmon config file (MITRE ATT&CK mapped)

```powershell
Invoke-WebRequest -Uri "https://wazuh.com/resources/blog/emulation-of-attack-techniques-and-detection-with-wazuh/sysmonconfig.xml" -OutFile "C:\Tools\Sysmon\sysmonconfig.xml"
```

---

### 2.6 — Go into the Sysmon folder

```powershell
cd C:\Tools\Sysmon
```

---

### 2.7 — Install Sysmon with the config file

```powershell
.\Sysmon64.exe -accepteula -i sysmonconfig.xml
```

Expected output:
```
Sysmon64 installed.
SysmonDrv installed.
Starting SysmonDrv.
SysmonDrv started.
Starting Sysmon64.
Sysmon64 started.
```

---

### 2.8 — Verify Sysmon is running

```powershell
Get-Service Sysmon64
```

Expected output:
```
Status   Name       DisplayName
------   ----       -----------
Running  Sysmon64   Sysmon64
```

✅ Status = `Running` means Sysmon is installed correctly.

---

### 2.9 — Verify events in Event Viewer

1. Press `Windows key + R` → type `eventvwr.msc` → Enter
2. Expand: **Applications and Services Logs → Microsoft → Windows → Sysmon → Operational**
3. You should see events appearing

---

# STEP 3 — Configure Windows Agent to Forward Sysmon Events

> **Where:** Windows machine (Windows Agent) — edit ossec.conf in Notepad as Administrator  
> **What this does:** Tells the Wazuh Agent to read Sysmon events and send them to the Manager

### 3.1 — Open the Wazuh Agent config file

Open **Notepad as Administrator**:
- `Windows key` → type `Notepad` → Right-click → **Run as administrator**
- **File → Open** → navigate to: `C:\Program Files (x86)\ossec-agent\`
- Change the file type dropdown to **All Files (*.*)**
- Open `ossec.conf`

---

### 3.2 — Add the Sysmon localfile block

Find `</ossec_config>` at the bottom. Paste this block **just BEFORE** it:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

**What each line means:**

| Tag             | Meaning                                                                 |
|-----------------|-------------------------------------------------------------------------|
| `<location>`    | The exact Windows Event Log channel where Sysmon writes events          |
| `<log_format>`  | Tells Wazuh to read Windows Event Log format                            |

---

### 3.3 — Save the file

**File → Save**

---

### 3.4 — Restart the Wazuh Agent on Windows

```powershell
Restart-Service -Name wazuh
```

---

### 3.5 — Verify the agent is running

```powershell
Get-Service wazuh
```

Expected output:
```
Status   Name    DisplayName
------   ----    -----------
Running  wazuh   Wazuh
```

---

### 3.6 — Verify Sysmon events are arriving at the Manager

On **Wazuh Manager**:
```bash
sudo tail -f /var/ossec/logs/archives/archives.log | grep -i sysmon
```
Move your mouse or open a program on Windows. Sysmon events should appear within seconds. Press `Ctrl+C` to stop.

---

# STEP 4 — Install Atomic Red Team on Windows

> **Where:** Windows machine — PowerShell as Administrator  
> **What this does:** Installs Atomic Red Team (ART) so you can simulate real ATT&CK attacks

> ⚠️ **WARNING:** Only do this on an isolated/disposable virtual machine. These simulate real attacks.

### 4.1 — Set PowerShell execution policy

```powershell
Set-ExecutionPolicy Bypass -Scope CurrentUser -Force
```

---

### 4.2 — Enable TLS 1.2 (required for download)

```powershell
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
```

---

### 4.3 — Install Atomic Red Team (module + all atomics)

```powershell
IEX (IWR 'https://raw.githubusercontent.com/redcanaryco/invoke-atomicredteam/master/install-atomicredteam.ps1' -UseBasicParsing); Install-AtomicRedTeam -getAtomics
```

This downloads everything to `C:\AtomicRedTeam\`. May take a few minutes.

---

### 4.4 — Import the ART module

```powershell
Import-Module "C:\AtomicRedTeam\invoke-atomicredteam\Invoke-AtomicRedTeam.psd1" -Force
```

> ⚠️ You must run this import command **every time** you open a new PowerShell window.

---

### 4.5 — Verify ART is working

```powershell
Invoke-AtomicTest T1053.005 -ShowDetailsBrief
```

✅ You should see a list of available sub-tests for T1053.005.

---

# STEP 5 — Add Detection Rules on Wazuh Manager

> **Where:** SSH into your Wazuh Manager (Linux machine)  
> **What this does:** Creates rules that generate alerts when Sysmon detects ATT&CK techniques

### 5.1 — Open the local rules file

```bash
sudo nano /var/ossec/etc/rules/local_rules.xml
```

---

### 5.2 — Add the 5 detection rules

Paste this entire block inside the file (after any existing content):

```xml
<group name="windows,sysmon,mitre,">

  <!-- Rule 1: T1053.005 - Scheduled Task -->
  <rule id="115001" level="10">
    <if_group>windows</if_group>
    <field name="win.eventdata.ruleName" type="pcre2">technique_id=T1053,technique_name=Scheduled Task</field>
    <description>MITRE T1053.005 - Scheduled Task created on $(win.system.computer)</description>
    <mitre>
      <id>T1053</id>
    </mitre>
  </rule>

  <!-- Rule 2: T1574.001 - DLL Side-Loading -->
  <rule id="115002" level="10">
    <if_group>windows</if_group>
    <field name="win.eventdata.ruleName" type="pcre2">technique_id=T1574.002,technique_name=DLL Side-Loading</field>
    <description>MITRE T1574.001 - DLL Side-Loading detected on $(win.system.computer)</description>
    <mitre>
      <id>T1574</id>
    </mitre>
  </rule>

  <!-- Rule 3: T1218.010 - Regsvr32 -->
  <rule id="115003" level="10">
    <if_group>windows</if_group>
    <field name="win.eventdata.ruleName" type="pcre2">technique_id=T1218.010,technique_name=Regsvr32</field>
    <description>MITRE T1218.010 - Regsvr32 Proxy Execution on $(win.system.computer)</description>
    <mitre>
      <id>T1218</id>
    </mitre>
  </rule>

  <!-- Rule 4: T1518.001 - Security Software Discovery -->
  <rule id="115004" level="10">
    <if_group>windows</if_group>
    <field name="win.eventdata.ruleName" type="pcre2">technique_id=T1518.001,technique_name=Security Software Discovery</field>
    <description>MITRE T1518.001 - Security Software Discovery on $(win.system.computer)</description>
    <mitre>
      <id>T1518</id>
    </mitre>
  </rule>

  <!-- Rule 5: T1548.002 - Bypass UAC (high severity = level 12) -->
  <rule id="115005" level="12">
    <if_group>windows</if_group>
    <field name="win.eventdata.ruleName" type="pcre2">technique_id=T1548.002,technique_name=Bypass User Access Control</field>
    <description>MITRE T1548.002 - UAC Bypass / Privilege Escalation on $(win.system.computer)</description>
    <mitre>
      <id>T1548.002</id>
    </mitre>
  </rule>

</group>
```

---

### 5.3 — Save the file

Press `Ctrl + X` → `Y` → `Enter`

---

### 5.4 — Restart Manager to load the new rules

```bash
sudo systemctl restart wazuh-manager
```

---

### 5.5 — Check for rule errors

```bash
sudo grep -i "error" /var/ossec/logs/ossec.log | tail -10
```

✅ No errors related to your rules = you are good to proceed.

---

# STEP 6 — Run ATT&CK Emulation Tests

> **Where:** Windows PowerShell + watch alerts.log on Manager simultaneously  
> **What this does:** Simulates real ATT&CK attacks so Wazuh can detect them

> ℹ️ Open **two windows**:
> - **Window 1:** PowerShell on Windows (to run tests)
> - **Window 2:** SSH on Manager — run: `sudo tail -f /var/ossec/logs/alerts/alerts.log`

---

### 6.1 — Import ART module (do this first in every new PS session)

```powershell
Import-Module "C:\AtomicRedTeam\invoke-atomicredteam\Invoke-AtomicRedTeam.psd1" -Force
```

---

### 6.2 — Test 1: T1053.005 — Scheduled Task

**Simulates:** Attacker creating a scheduled task for persistence

```powershell
# Run the test
Invoke-AtomicTest T1053.005

# Wait 10–15 seconds → expect alert with rule id 115001 on Manager

# Clean up
Invoke-AtomicTest T1053.005 -Cleanup
```

---

### 6.3 — Test 2: T1218.010 — Regsvr32

**Simulates:** Using a legitimate signed Windows binary to execute malicious code

```powershell
Invoke-AtomicTest T1218.010 -CheckPrereqs
Invoke-AtomicTest T1218.010

# expect rule id 115003

Invoke-AtomicTest T1218.010 -Cleanup
```

---

### 6.4 — Test 3: T1518.001 — Security Software Discovery

**Simulates:** Attacker discovering what antivirus/security tools are installed

```powershell
Invoke-AtomicTest T1518.001

# expect rule id 115004

Invoke-AtomicTest T1518.001 -Cleanup
```

---

### 6.5 — Test 4: T1548.002 — Bypass UAC

**Simulates:** Bypassing User Account Control to escalate privileges

```powershell
Invoke-AtomicTest T1548.002 -GetPrereqs
Invoke-AtomicTest T1548.002

# expect rule id 115005 (level 12 = high severity)

Invoke-AtomicTest T1548.002 -Cleanup
```

---

### 6.6 — Test 5: T1574.001 — DLL Side-Loading

**Simulates:** Attacker hijacking a DLL loaded by a legitimate application

```powershell
Invoke-AtomicTest T1574.001

# expect rule id 115002

Invoke-AtomicTest T1574.001 -Cleanup
```

---

# STEP 7 — Verify Alerts on Wazuh Dashboard

> **Where:** Browser — open the Wazuh Dashboard

### 7.1 — Open the Dashboard

```
https://MANAGER_PRIVATE_IP
```
Log in with your admin credentials.

---

### 7.2 — Go to Security Events

Left sidebar → **Threat Intelligence** → **Events**  
Set time range (top right) to **Today** or **Last 1 hour**

---

### 7.3 — Search for your alerts by rule ID

| Search Query            | What it finds                  |
|-------------------------|--------------------------------|
| `rule.id: 115001`       | Scheduled Task alert           |
| `rule.id: 115002`       | DLL Side-Loading               |
| `rule.id: 115003`       | Regsvr32                       |
| `rule.id: 115004`       | Security Software Discovery    |
| `rule.id: 115005`       | UAC Bypass (high severity)     |

---

### 7.4 — Search by MITRE technique

```
rule.mitre.id: T1548.002
```

---

### 7.5 — See all Sysmon alerts together

```
rule.groups: sysmon
```

---

### 7.6 — View MITRE ATT&CK matrix

Left sidebar → **Threat Intelligence** → **MITRE ATT&CK**  
Detected techniques will be highlighted on the ATT&CK matrix.

✅ All 5 techniques should appear highlighted.

---

# TROUBLESHOOTING

### Problem: No syslog alerts on Manager

```bash
sudo ss -tlnp | grep 514
```
If nothing shows, the `<remote>` block has an error. Re-check `ossec.conf` and restart.  
Also check that `allowed-ips` matches your Linux Agent's IP exactly.

---

### Problem: Sysmon events not arriving at Manager

On Windows, check Agent logs:
```powershell
type "C:\Program Files (x86)\ossec-agent\ossec.log" | Select-Object -Last 30
```

Check Agent is running:
```powershell
Get-Service wazuh
```

If stopped, start it:
```powershell
Start-Service wazuh
```

---

### Problem: ART test runs but no alert appears

Wait 30 seconds first. Then confirm:
1. Agent is running (Step 3.5)
2. Rules were loaded (Step 5.4 — restart Manager)
3. Sysmon config maps MITRE technique IDs in the `ruleName` field

Check events are arriving at all:
```bash
sudo tail -f /var/ossec/logs/archives/archives.log | grep -i "sysmon\|windows"
```

---

### Problem: Rules not triggering

Test a rule manually on the Manager:
```bash
sudo /var/ossec/bin/wazuh-logtest
```
Paste a Sysmon event log line. It will tell you whether the rule matches or not.

---

# KEY FILE PATHS — Quick Reference

### Wazuh Manager (Linux)

| File                                          | Purpose                                        |
|-----------------------------------------------|------------------------------------------------|
| `/var/ossec/etc/ossec.conf`                   | Main config — add syslog `<remote>` block here |
| `/var/ossec/etc/rules/local_rules.xml`        | Add custom detection rules here                |
| `/var/ossec/logs/alerts/alerts.log`           | All alerts — watch this during tests           |
| `/var/ossec/logs/archives/archives.log`       | All received events (including non-alerts)     |
| `/var/ossec/logs/ossec.log`                   | Manager error and info logs                    |

### Windows Agent

| File                                                        | Purpose                                    |
|-------------------------------------------------------------|--------------------------------------------|
| `C:\Program Files (x86)\ossec-agent\ossec.conf`             | Agent config — add Sysmon `<localfile>` here |
| `C:\Program Files (x86)\ossec-agent\ossec.log`              | Agent error and info logs                  |
| `C:\AtomicRedTeam\atomics\`                                  | All ART test definitions                   |
| `C:\AtomicRedTeam\invoke-atomicredteam\`                    | ART PowerShell module                      |
| `C:\Tools\Sysmon\`                                          | Sysmon binary and config                   |

---

# ESSENTIAL COMMANDS — Quick Reference

### Wazuh Manager (Linux)

```bash
sudo systemctl restart wazuh-manager            # restart manager
sudo systemctl status wazuh-manager             # check status
sudo tail -f /var/ossec/logs/alerts/alerts.log  # watch alerts live
sudo ss -tlnp | grep 514                        # verify syslog port open
```

### Windows Agent (PowerShell as Admin)

```powershell
Restart-Service -Name wazuh                     # restart agent
Get-Service wazuh                               # check agent status
Get-Service Sysmon64                            # check Sysmon status
Import-Module "C:\AtomicRedTeam\invoke-atomicredteam\Invoke-AtomicRedTeam.psd1" -Force
Invoke-AtomicTest T1053.005                     # run a test
Invoke-AtomicTest T1053.005 -Cleanup            # clean up after test
Invoke-AtomicTest T1053.005 -ShowDetailsBrief   # list available tests
```

---

# TEST RESULT CHECKLIST

Use this after completing the lab to verify everything worked.

| #  | Check                                         | Expected Result                              | Status |
|----|-----------------------------------------------|----------------------------------------------|--------|
| 1  | Port 514 listening on Manager                 | `wazuh-remoted` visible in `ss` output       | ☐      |
| 2  | Test syslog received from Linux Agent         | Message visible in `archives.log`            | ☐      |
| 3  | Sysmon service running on Windows             | `Get-Service Sysmon64` → `Running`           | ☐      |
| 4  | Sysmon events visible in Event Viewer         | Events in Sysmon/Operational channel         | ☐      |
| 5  | Wazuh Agent running on Windows                | `Get-Service wazuh` → `Running`              | ☐      |
| 6  | Sysmon events arriving at Manager             | Output visible in `archives.log`             | ☐      |
| 7  | ART module imports without error              | No error after `Import-Module`               | ☐      |
| 8  | T1053.005 alert triggered                     | Rule `115001` visible in `alerts.log`        | ☐      |
| 9  | T1218.010 alert triggered                     | Rule `115003` visible in `alerts.log`        | ☐      |
| 10 | T1518.001 alert triggered                     | Rule `115004` visible in `alerts.log`        | ☐      |
| 11 | T1548.002 alert triggered                     | Rule `115005` visible in `alerts.log`        | ☐      |
| 12 | T1574.001 alert triggered                     | Rule `115002` visible in `alerts.log`        | ☐      |
| 13 | All alerts visible in Wazuh Dashboard         | Events found under Threat Intelligence       | ☐      |
| 14 | MITRE ATT&CK matrix shows detections          | 5 techniques highlighted on matrix           | ☐      |

---

*Sources: Wazuh Documentation (syslog), Wazuh Blog (Sysmon monitoring), Wazuh Blog (ATT&CK emulation with ART)*
