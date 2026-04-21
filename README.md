# 🛡️ Home SIEM Lab — Splunk + Sysmon + Kali Linux
 
> A fully functional Security Information and Event Management (SIEM) lab built on VirtualBox, simulating real-world SOC operations through endpoint telemetry collection, log forwarding, and threat detection using Splunk Enterprise.
 
---
 
## 📌 Table of Contents
 
- [Overview](#-overview)
- [Lab Architecture](#-lab-architecture)
- [Environment & Tools](#-environment--tools)
- [Setup Walkthrough](#-setup-walkthrough)
  - [1. Virtualization](#1-virtualization)
  - [2. Network Configuration](#2-network-configuration)
  - [3. Splunk Server — Ubuntu](#3-splunk-server--ubuntu)
  - [4. Windows Log Forwarding](#4-windows-log-forwarding)
  - [5. Log Collection Configuration](#5-log-collection-configuration)
  - [6. Sysmon Deployment](#6-sysmon-deployment)
- [Validation & SPL Queries](#-validation--spl-queries)
- [Troubleshooting & Key Learnings](#-troubleshooting--key-learnings)
- [Skills Demonstrated](#-skills-demonstrated)
- [Next Steps](#-next-steps)
---
 
## 🧠 Overview
 
This project simulates a lightweight Security Operations Center (SOC) environment using free and open-source tools. The goal was to:
 
- Deploy a SIEM platform (Splunk Enterprise) on a Linux server
- Forward real Windows endpoint logs using Splunk Universal Forwarder
- Enrich telemetry with Sysmon for deep process and network visibility
- Validate the full pipeline with SPL (Search Processing Language)
- Use Kali Linux as a simulated attacker for future detection engineering
This lab directly supports preparation for the **Splunk Core Power User (SPLK-2003)** certification.
 
---
 
## 🏗️ Lab Architecture
 
```
┌─────────────────────────────────────────────────────┐
│                   VirtualBox Host                    │
│                                                     │
│  ┌──────────────┐        ┌──────────────────────┐   │
│  │  Kali Linux  │        │     Windows 10        │   │
│  │  (Attacker)  │───────▶│  (Target / Log Source)│   │
│  │              │  Attack│                       │   │
│  └──────────────┘        │  ┌─────────────────┐  │   │
│   192.168.56.103         │  │ Sysmon          │  │   │
│                          │  │ UF (port 9997)  │  │   │
│                          │  └────────┬────────┘  │   │
│                          └───────────┼───────────┘   │
│                           192.168.56.101             │
│                                      │               │
│                                      ▼               │
│                          ┌───────────────────────┐   │
│                          │    Ubuntu 22.04 LTS    │   │
│                          │   Splunk Enterprise    │   │
│                          │   Web UI :8000         │   │
│                          │   Receive :9997        │   │
│                          └───────────────────────┘   │
│                            192.168.56.102             │
│                                                     │
│  Network: Adapter 1 = NAT (Internet)                │
│           Adapter 2 = Host-Only (Internal Lab)      │
└─────────────────────────────────────────────────────┘
```
 
---
 
## 🖥️ Environment & Tools
 
| Component | Details |
|---|---|
| **Hypervisor** | VirtualBox (free) |
| **SIEM** | Splunk Enterprise 10.2 (free trial / 500MB/day) |
| **Log Forwarder** | Splunk Universal Forwarder |
| **Endpoint Telemetry** | Sysmon (SwiftOnSecurity config) |
| **Attacker Machine** | Kali Linux Rolling |
| **Target/Log Source** | Windows 10 Enterprise (Eval) |
| **SIEM Server OS** | Ubuntu Server 22.04 LTS |
 
### VM Resource Allocation
 
| VM | RAM | Storage | IP |
|---|---|---|---|
| Ubuntu (Splunk) | 4 GB | 50 GB | `192.168.56.102` |
| Windows 10 | 2 GB | 40 GB | `192.168.56.101` |
| Kali Linux | 2 GB | 30 GB | `192.168.56.103` |
 
---
 
## ⚙️ Setup Walkthrough
 
### 1. Virtualization
 
- Installed **VirtualBox** on host laptop
- Created 3 VMs: Ubuntu Server, Windows 10, Kali Linux
- Configured each with dual network adapters for internet + internal lab communication
### 2. Network Configuration
 
Each VM was configured with:
 
- **Adapter 1 (NAT)** → Internet access (for downloads, updates)
- **Adapter 2 (Host-Only: `vboxnet0`)** → Internal lab communication
This isolates lab attack traffic from the real network while maintaining internet access.
 
```
Host-Only Network Range: 192.168.56.0/24
Ubuntu (Splunk):  192.168.56.102
Windows 10:       192.168.56.101
Kali Linux:       192.168.56.103
```
 
### 3. Splunk Server — Ubuntu
 
```bash
# Install Splunk Enterprise
sudo dpkg -i splunk-*.deb
 
# Start and accept license
sudo /opt/splunk/bin/splunk start --accept-license
 
# Enable receiving port for Universal Forwarder
/opt/splunk/bin/splunk enable listen 9997
 
# Enable auto-start on boot
sudo /opt/splunk/bin/splunk enable boot-start
```
 
**Access Web UI:** `http://192.168.56.102:8000`
 
**Firewall rules configured:**
 
```bash
sudo ufw allow 22/tcp     # SSH
sudo ufw allow 8000/tcp   # Splunk Web UI
sudo ufw allow 9997/tcp   # Forwarder ingestion
sudo ufw allow 8089/tcp   # Splunk REST API
sudo ufw enable
```
 
### 4. Windows Log Forwarding
 
- Downloaded and installed **Splunk Universal Forwarder** on Windows 10
- During install, pointed forwarder to Splunk server:
```
Receiving Indexer: 192.168.56.102:9997
Run As: NT AUTHORITY\SYSTEM (Local System)
```
 
### 5. Log Collection Configuration
 
Created `inputs.conf` on the Windows forwarder:
 
```ini
[WinEventLog://Security]
disabled = 0
 
[WinEventLog://System]
disabled = 0
 
[WinEventLog://Application]
disabled = 0
 
[WinEventLog://Microsoft-Windows-Sysmon/Operational]
disabled = 0
start_from = oldest
current_only = 0
renderXml = true
```
 
**File location:** `C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf`
 
Restart forwarder after changes:
 
```powershell
Restart-Service SplunkForwarder
```
 
### 6. Sysmon Deployment
 
Installed **Sysmon** with the [SwiftOnSecurity config](https://github.com/SwiftOnSecurity/sysmon-config) for maximum telemetry:
 
```powershell
# Install Sysmon with config
sysmon64.exe -accepteula -i sysmonconfig-export.xml
```
 
**Sysmon provides visibility into:**
 
| Event ID | Description |
|---|---|
| `1` | Process Creation |
| `3` | Network Connection |
| `7` | Image/DLL Loaded |
| `11` | File Created |
| `13` | Registry Value Set |
| `22` | DNS Query |
 
---
 
## 📊 Validation & SPL Queries
 
### Verify All Data Sources Are Flowing
 
```spl
index=main
| stats count by host, source, sourcetype
```
 
### Sysmon Process Launched
 
```spl
index=main source="WinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=1
``` 
---
 
## 🔥 Troubleshooting & Key Learnings
 
| Problem | Root Cause | Fix |
|---|---|---|
| VMs couldn't communicate | No internal network | Added Host-Only Adapter 2 to all VMs |
| Forwarder showed "inactive" | Wrong IP in `outputs.conf` | Corrected to `192.168.56.102:9997` |
| Port 9997 not receiving | Splunk listener not enabled | Ran `splunk enable listen 9997` |
| Sysmon logs not appearing | Wrong event channel name | Used exact name `Microsoft-Windows-Sysmon/Operational` |
| Sysmon access denied | Forwarder ran as limited user | Set forwarder to run as `NT AUTHORITY\SYSTEM` |
 
**Diagnostic tools used:**
 
```bash
# Validate Splunk config (run on forwarder)
splunk btool inputs list --debug
 
# Test connectivity from Windows to Splunk
Test-NetConnection -ComputerName 192.168.56.102 -Port 9997
 
# Check forwarder logs
Get-Content "C:\Program Files\SplunkUniversalForwarder\var\log\splunk\splunkd.log" -Tail 50
```
 
---
 
## 🎯 Skills Demonstrated
 
- **SIEM Deployment** — Splunk Enterprise installation, configuration, and administration on Linux
- **Log Engineering** — `inputs.conf`, `outputs.conf`, sourcetype configuration, index management
- **Endpoint Telemetry** — Sysmon deployment and Sysinternals tooling
- **Network Architecture** — Dual-adapter VM networking, firewall rules, static IP configuration
- **SPL (Search Processing Language)** — `stats`, `eval`, `bin`, `table`, `where` commands
- **Troubleshooting** — `btool` config validation, connectivity testing, log analysis
- **Linux Administration** — Ubuntu Server setup, `ufw`, `systemctl`, `netplan`
- **Security Concepts** — Log forwarding pipeline, SOC workflows, endpoint detection
---
 
## 🚀 Next Steps
 
- [ ] Simulate attacks from Kali Linux (brute force, port scan, reverse shell)
- [ ] Build Splunk dashboards for login failures and process anomalies
- [ ] Write MITRE ATT&CK mapped detection rules (T1078, T1059, T1110)
- [ ] Add pfSense VM for network-level firewall logging
- [ ] Complete **Splunk Core Power User (SPLK-2003)** certification
- [ ] Integrate **Boss of the SOC (BOTS)** dataset for threat hunting practice
---
 
## 📚 References
 
- [Splunk Documentation](https://docs.splunk.com)
- [Splunk Education — Free Fundamentals](https://education.splunk.com)
- [SwiftOnSecurity Sysmon Config](https://github.com/SwiftOnSecurity/sysmon-config)
- [Boss of the SOC (BOTS) Dataset](https://github.com/splunk/botsv3)
- [Microsoft Sysinternals — Sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon)
- [MITRE ATT&CK Framework](https://attack.mitre.org)
---
 
*Built as part of hands-on preparation for the Splunk Core Power User certification and SOC Analyst career track.*
 
