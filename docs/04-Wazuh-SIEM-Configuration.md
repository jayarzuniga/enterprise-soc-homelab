# 1. Introduction to Wazuh

## 1.1 What is Wazuh?

Wazuh is a free, open-source security monitoring platform that provides:

- Security Information and Event Management (SIEM)
- Intrusion Detection System (IDS)
- Log analysis and correlation
- File integrity monitoring (FIM)
- Vulnerability detection
- Configuration assessment
- Incident response capabilities
- Compliance monitoring (PCI DSS, GDPR, HIPAA, etc.)

---

## 1.2 Why Use Wazuh in a Homelab?

Wazuh transforms your homelab into a Security Operations Center (SOC) by:

- Providing real-time security alerts for suspicious activities
- Monitoring file changes on critical systems
- Detecting malware and rootkits
- Tracking user authentication and system access
- Analyzing logs from multiple systems in one place
- Teaching professional cybersecurity skills
- Simulating enterprise security infrastructure

---

# 2. System Architecture

## 2.1 Wazuh Components

A Wazuh deployment consists of three main components:

**1. Wazuh Manager (Server)** — The central server that receives and analyzes data from agents, stores security events, processes rules and decoders, and manages agent connections.

**2. Wazuh Agents (Clients)** — Lightweight software installed on monitored systems that collect logs, monitor file integrity, detect rootkits, and send data to the manager. Runs on Windows, Linux, and macOS.

**3. Wazuh Dashboard (Optional)** — Web interface for viewing alerts, creating visualizations, and managing agents and rules. *(Not covered in this guide — focuses on manager-agent setup.)*

---

## 2.2 Communication Flow

| Step | Description                                              |
| ---- | -------------------------------------------------------- |
| 1    | Agent collects security events (logs, file changes)      |
| 2    | Agent encrypts and compresses data                       |
| 3    | Agent sends data to Manager via **TCP port 1514**        |
| 4    | Manager decrypts and analyzes data                       |
| 5    | Manager applies rules and generates alerts               |
| 6    | Alerts are stored and forwarded to dashboard/SIEM        |

**Key Ports:**

| Port     | Protocol | Purpose                              |
| -------- | -------- | ------------------------------------ |
| `1514`   | TCP      | Agent–Manager communication (encrypted) |
| `1515`   | TCP      | Agent enrollment (authentication)    |
| `55000`  | TCP      | Wazuh API (dashboard integration)    |

---

## 2.3 Lab Topology

> ⚠️ **My Lab Note:** The ideal topology uses separate subnets per network segment. Due to limited resources, all systems in my lab are placed on the `192.168.3.x/24` network. The IPs below reflect my actual setup.

| System                   | Role              | IP Address       | OS                      |
| ------------------------ | ----------------- | ---------------- | ----------------------- |
| Wazuh Manager            | SIEM / Manager    | 192.168.3.81     | Ubuntu 22.04            |
| Windows Server (DC01)    | Wazuh Agent (AD)  | 192.168.3.10     | Windows Server 2022     |
| Windows Workstation      | Wazuh Agent       | 192.168.3.20     | Windows Enterprise      |
| Kali Linux               | Attacker VM       | 192.168.3.30     | Kali Linux              |

**Network Flow:**

```
Windows Server (3.10) ──┐
Windows Workstation (3.20) ─┼──→ TCP 1514 ──→ Wazuh Manager (3.81)
Kali Linux (3.30) ──────┘
```

> ⚠️ All systems must be able to reach the Wazuh Manager on **TCP port 1514**. Ensure pfSense firewall rules permit this traffic.

---

# 3. Prerequisites

## 3.1 System Requirements

**Wazuh Manager (Ubuntu VM):**

| Component | Minimum  | Recommended                  |
| --------- | -------- | ---------------------------- |
| OS        | Ubuntu 22.04 LTS | Ubuntu 22.04 LTS   |
| RAM       | 2 GB     | 4 GB                         |
| Disk      | 20 GB    | 50 GB *(for log storage)*    |
| CPU       | 2 cores  | 2+ cores                     |
| Network   | Static IP configured | —            |

**Wazuh Agents (Windows / Linux):**

| Component | Requirement                              |
| --------- | ---------------------------------------- |
| RAM       | ~50–100 MB per agent                     |
| Disk      | ~100 MB per agent                        |
| Network   | Must reach Manager on TCP 1514 and 1515  |

---

## 3.2 Pre-Installation Checklist

- [ ] Wazuh Manager VM is running Ubuntu 22.04 LTS
- [ ] Manager has a static IP configured (`192.168.3.81`)
- [ ] Windows Server (DC01) is set up and joined to the domain
- [ ] Windows Workstation is set up with Windows Enterprise
- [ ] All VMs have internet access for package downloads
- [ ] You have `sudo`/Administrator access on all systems
- [ ] pfSense firewall allows TCP 1514 and 1515 from all agent IPs to `192.168.3.81`

---

## 3.3 Version Compatibility

> ⚠️ **Important:** The agent version must be **equal to or lower than** the manager version. This guide uses **Wazuh 4.7.5** for all components.

| Risk                      | Description                                              |
| ------------------------- | -------------------------------------------------------- |
| Newer agent on older manager | Protocol incompatibilities prevent communication      |
| Version mismatch          | Security vulnerabilities and unexpected behavior         |

---

# 4. Wazuh Manager Installation (Ubuntu VM)

## 4.1 Add Wazuh Repository

**Step 1 — Download and install the Wazuh GPG key:**

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --dearmor -o /usr/share/keyrings/wazuh.gpg
```

This downloads Wazuh's official signing key and converts it to binary format so Ubuntu can verify package authenticity.

**Step 2 — Add the Wazuh repository:**

```bash
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list
```

**Step 3 — Update package lists:**

```bash
sudo apt update
```

Expected output: `Reading package lists... Done` with no errors.

---

## 4.2 Install Wazuh Manager

```bash
sudo apt install wazuh-manager=4.7.5-1 -y
```

This installs the manager daemon, configuration files under `/var/ossec/`, rule files, and the systemd service. Expect ~200–300 MB download and 2–5 minutes to complete.

---

## 4.3 Verify Installation

```bash
/var/ossec/bin/wazuh-managerd -V
```

> ✅ Expected output: `Wazuh v4.7.5`

---

## 4.4 Start and Enable Wazuh Manager

```bash
sudo systemctl enable wazuh-manager
sudo systemctl start wazuh-manager
sudo systemctl status wazuh-manager
```

| Status indicator          | Meaning                         |
| ------------------------- | ------------------------------- |
| `Active: active (running)`| ✅ Running correctly             |
| `Active: failed`          | ❌ Crashed or failed to start   |
| `Inactive: dead`          | ⚠️ Stopped                      |
| `Enabled`                 | ✅ Will start on boot            |

Press `Q` to exit the status view.

---

## 4.5 Verify Manager is Listening for Agents

```bash
sudo ss -tulnp | grep 1514
```

> ✅ Expected output: `tcp LISTEN 0.0.0.0:1514 ... ossec-remoted`

> ⚠️ If you see `127.0.0.1:1514` instead of `0.0.0.0:1514`, agents **cannot** connect. Check `/var/ossec/etc/ossec.conf` and ensure the `<remote>` section uses `<connection>secure</connection>`.

---

## 4.6 Configure Firewall

```bash
sudo ufw allow 1514/tcp
sudo ufw allow 1515/tcp
sudo ufw reload
sudo ufw status numbered
```

Should show both `1514/tcp` and `1515/tcp` as `ALLOW IN`.

> ✅ **Wazuh Manager installation complete!**

---

# 5. Wazuh Agent Installation — Linux (Ubuntu)

> **Note:** This section covers Linux agents. For Windows agents (DC01 and Workstation), see **Section 6**.

## 5.1 Remove Old Agent Installation *(If Exists)*

```bash
sudo systemctl stop wazuh-agent
sudo apt remove wazuh-agent -y
sudo rm -rf /var/ossec
```

> ⚠️ This deletes all Wazuh data on this system including logs, configs, and encryption keys. Only run if you want a completely clean installation.

---

## 5.2 Add Wazuh Repository *(If Not Already Added)*

Run the same repository setup commands from **Section 4.1** on the agent VM.

---

## 5.3 Install Wazuh Agent

```bash
sudo apt install wazuh-agent=4.7.5-1 -y
```

**Prevent automatic version upgrades:**

```bash
sudo apt-mark hold wazuh-agent
```

> ⚠️ Always hold the agent version to prevent automatic upgrades that could break compatibility with the manager.

---

## 5.4 Configure Agent to Connect to Manager

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Find the `<client>` section and configure:

```xml
<client>
  <server>
    <address>192.168.3.81</address>
    <port>1514</port>
    <protocol>tcp</protocol>
  </server>
</client>
```

Save and exit: `Ctrl+X` → `Y` → `Enter`

---

## 5.5 Start and Enable Agent Service

```bash
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
sudo systemctl status wazuh-agent
```

> ⚠️ The agent will **not** successfully connect yet — it needs to be registered first (see **Section 7**).

---

# 6. Wazuh Agent Installation — Windows (DC01 & Workstation)

This section covers installing the Wazuh agent on both:
- **Windows Server 2022** — `DC01` at `192.168.3.10`
- **Windows Enterprise Workstation** at `192.168.3.20`

Repeat all steps in this section for each Windows machine.

## 6.1 Download Wazuh Agent

On each Windows machine, open a browser and download:

```
https://packages.wazuh.com/4.x/windows/wazuh-agent-4.7.5-1.msi
```

Save the `.msi` file to the Downloads folder.

---

## 6.2 Install Wazuh Agent

1. Double-click `wazuh-agent-4.7.5-1.msi`
2. Click **Next** on the welcome screen
3. Configure agent settings:

   | Setting              | DC01             | Workstation      |
   | -------------------- | ---------------- | ---------------- |
   | Wazuh server address | `192.168.3.81`   | `192.168.3.81`   |
   | Wazuh server port    | `1514`           | `1514`           |
   | Protocol             | TCP              | TCP              |
   | Agent name           | `DC01`           | `WS01`           |

4. Click **Next** → **Install** → **Finish**

---

## 6.3 Register Agent via PowerShell

Open **PowerShell as Administrator** on each Windows machine:

```powershell
cd "C:\Program Files (x86)\ossec-agent"
.\agent-auth.exe -m 192.168.3.81
```

Expected output:

```
INFO: Connected to enrollment service.
INFO: Using agent name as: DC01
INFO: Valid key received.
INFO: Agent successfully authenticated.
```

**Start the Wazuh service:**

```powershell
NET START WazuhSvc
```

---

## 6.4 Verify Windows Agent is Running

```powershell
Get-Service WazuhSvc
```

Expected: `Status = Running`

**Test connectivity to the manager:**

```powershell
Test-NetConnection 192.168.3.81 -Port 1514
```

Expected: `TcpTestSucceeded: True`

---

# 7. Agent Registration and Authentication

## 7.1 Check for Duplicate Agent Names *(On Manager)*

Before registering, check if an old entry already exists:

```bash
sudo /var/ossec/bin/manage_agents
```

Type `3` to list agents. If a duplicate name exists:
1. Type `4` to remove it, enter the agent ID, confirm
2. Type `5` to quit

---

## 7.2 Authenticate Linux Agents *(On Agent VM)*

```bash
sudo /var/ossec/bin/agent-auth -m 192.168.3.81
```

Expected output:

```
INFO: Connected to enrollment service.
INFO: Using agent name as: <hostname>
INFO: Valid key received.
INFO: Agent successfully authenticated.
```

---

## 7.3 Verify All Agents Are Connected *(On Manager)*

```bash
sudo /var/ossec/bin/agent_control -lc
```

You should see all registered agents listed as **Active**:

```
ID: 001, Name: DC01,  IP: 192.168.3.10, Active/Online
ID: 002, Name: WS01,  IP: 192.168.3.20, Active/Online
```

> ✅ All agents are now sending logs to Wazuh Manager.

---

## 7.4 Check Agent Logs *(On Agent)*

**Linux:**

```bash
sudo tail -f /var/ossec/logs/ossec.log
```

**Windows (PowerShell):**

```powershell
Get-Content "C:\Program Files (x86)\ossec-agent\ossec.log" -Tail 50
```

Look for:

```
INFO: Connected to the server (192.168.3.81:1514/tcp).
INFO: Agent is now online.
```

---

# 8. Verification and Testing

## 8.1 Complete System Check

**On Manager VM:**

```bash
sudo systemctl status wazuh-manager          # Service status
sudo ss -tulnp | grep 1514                   # Port listening
sudo /var/ossec/bin/agent_control -lc        # List active agents
sudo tail -50 /var/ossec/logs/ossec.log      # Manager logs
```

**On Linux Agent VMs:**

```bash
sudo systemctl status wazuh-agent
sudo tail -50 /var/ossec/logs/ossec.log
sudo cat /var/ossec/etc/client.keys          # Verify encryption key exists
```

**On Windows Agents (PowerShell):**

```powershell
Get-Service WazuhSvc
Get-Content "C:\Program Files (x86)\ossec-agent\ossec.log" -Tail 50
```

---

## 8.2 Test Alert Generation

**On any Linux agent, create and delete a test file:**

```bash
sudo touch /root/test_alert.txt
sudo rm /root/test_alert.txt
```

**On any Windows agent (PowerShell):**

```powershell
New-Item -Path "C:\test_alert.txt" -ItemType File
Remove-Item "C:\test_alert.txt"
```

**Check alerts on Manager:**

```bash
sudo tail -f /var/ossec/logs/alerts/alerts.log
```

> ✅ You should see file creation and deletion alerts for each test file — confirming all agents are sending data correctly.

---

# 9. Security Hardening

## 9.1 Manager Security

| Task                      | Command / Action                                             |
| ------------------------- | ------------------------------------------------------------ |
| Restrict by IP            | `sudo ufw allow from 192.168.3.0/24 to any port 1514`       |
| Backup encryption keys    | Back up `/var/ossec/etc/client.keys` regularly              |
| Monitor manager logs      | Review `/var/ossec/logs/` daily for suspicious activity      |
| Regular updates           | `sudo apt update && sudo apt list --upgradable` monthly      |

---

## 9.2 Agent Security

**Linux agents:**

```bash
sudo apt-mark hold wazuh-agent                          # Prevent auto-upgrades
sudo chmod 640 /var/ossec/etc/client.keys               # Protect encryption key
```

**Windows agents (PowerShell):**

```powershell
# Check held packages equivalent - pin version via installer only
# Verify key file permissions on:
# C:\Program Files (x86)\ossec-agent\client.keys
```

---

# 10. Monitoring and Maintenance

## 10.1 Daily Checks

```bash
sudo /var/ossec/bin/agent_control -lc                                  # All agents active?
sudo tail -100 /var/ossec/logs/alerts/alerts.log                       # Critical alerts
df -h /var/ossec                                                        # Disk space
sudo grep "authentication" /var/ossec/logs/ossec.log                   # Auth attempts
```

---

## 10.2 Weekly Tasks

- Review agent logs for errors on all systems
- Check manager CPU, RAM, and disk usage
- Verify log rotation: `ls -lh /var/ossec/logs/archives/`
- Test alert generation on each agent
- Review and tune alerting rules if needed

---

## 10.3 Monthly Tasks

- Check for Wazuh updates
- Audit agent list — remove any decommissioned agents
- Test backup and restore procedures
- Update documentation with any configuration changes

---

## 10.4 Log Management

Configure log rotation in the manager:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Find the `<global>` section and add:

```xml
<global>
  <logall>yes</logall>
  <logall_json>yes</logall_json>
  <email_notification>no</email_notification>
  <rotate_interval>daily</rotate_interval>
  <rotate_max_size>100M</rotate_max_size>
</global>
```

---

# 11. Troubleshooting Common Issues

## 11.1 Agent Shows "Disconnected" or "Never Connected"

| Cause                    | Fix                                                              |
| ------------------------ | ---------------------------------------------------------------- |
| Wrong manager IP         | Edit `ossec.conf`, verify `192.168.3.81`, restart agent         |
| Firewall blocking 1514   | `sudo ufw allow 1514/tcp && sudo ufw reload` on manager         |
| Manager not running      | `sudo systemctl status wazuh-manager`                           |
| Duplicate agent name     | Use `manage_agents` on manager to remove old entry              |
| Wrong encryption key     | Re-run `agent-auth -m 192.168.3.81` on the agent               |

---

## 11.2 "Agent Version Must Be Lower Than Manager"

```bash
/var/ossec/bin/wazuh-managerd -V      # Check manager version
/var/ossec/bin/wazuh-agentd -V        # Check agent version
sudo apt remove wazuh-agent -y
sudo apt install wazuh-agent=4.7.5-1 -y
sudo apt-mark hold wazuh-agent
```

---

## 11.3 Manager Only Listening on 127.0.0.1:1514

Edit the manager config:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Ensure the `<remote>` section looks like this:

```xml
<remote>
  <connection>secure</connection>
  <port>1514</port>
  <protocol>tcp</protocol>
</remote>
```

```bash
sudo systemctl restart wazuh-manager
sudo ss -tulnp | grep 1514            # Verify 0.0.0.0:1514
```

---

## 11.4 Windows Agent Not Connecting

```powershell
Get-Service WazuhSvc                                          # Service running?
Test-NetConnection 192.168.3.81 -Port 1514                    # Port reachable?
notepad "C:\Program Files (x86)\ossec-agent\ossec.conf"       # Correct IP?
```

If all looks correct, re-register:

```powershell
cd "C:\Program Files (x86)\ossec-agent"
.\agent-auth.exe -m 192.168.3.81
Restart-Service WazuhSvc
```

> Also verify **pfSense firewall rules** allow traffic from `192.168.3.x` to `192.168.3.81` on port `1514`.

---

# 12. Quick Reference

## 12.1 Manager Commands

```bash
# Service management
sudo systemctl status wazuh-manager
sudo systemctl restart wazuh-manager

# Agent management
sudo /var/ossec/bin/manage_agents           # Interactive menu
sudo /var/ossec/bin/agent_control -lc       # List connected agents

# Log viewing
sudo tail -f /var/ossec/logs/ossec.log
sudo tail -f /var/ossec/logs/alerts/alerts.log

# Check port
sudo ss -tulnp | grep 1514
```

## 12.2 Windows Agent Commands

```powershell
# Service management
Get-Service WazuhSvc
Restart-Service WazuhSvc
NET START WazuhSvc

# Authentication
cd "C:\Program Files (x86)\ossec-agent"
.\agent-auth.exe -m 192.168.3.81

# Log viewing
Get-Content "C:\Program Files (x86)\ossec-agent\ossec.log" -Tail 50
```

---

## 12.3 Important File Locations

| Component        | Path                                          | Purpose                    |
| ---------------- | --------------------------------------------- | -------------------------- |
| Manager (Linux)  | `/var/ossec/etc/ossec.conf`                   | Main configuration         |
| Manager (Linux)  | `/var/ossec/logs/ossec.log`                   | Manager logs               |
| Manager (Linux)  | `/var/ossec/logs/alerts/`                     | Alert logs                 |
| Manager (Linux)  | `/var/ossec/ruleset/`                         | Detection rules            |
| Manager (Linux)  | `/var/ossec/etc/client.keys`                  | Agent encryption keys      |
| Agent (Linux)    | `/var/ossec/etc/ossec.conf`                   | Agent configuration        |
| Agent (Linux)    | `/var/ossec/logs/ossec.log`                   | Agent logs                 |
| Agent (Windows)  | `C:\Program Files (x86)\ossec-agent\ossec.conf` | Agent configuration      |
| Agent (Windows)  | `C:\Program Files (x86)\ossec-agent\ossec.log`  | Agent logs               |
| Agent (Windows)  | `C:\Program Files (x86)\ossec-agent\client.keys`| Encryption key           |

---

## 12.4 Resources

- Official Documentation: https://documentation.wazuh.com
- GitHub Repository: https://github.com/wazuh/wazuh
- Community Forum: https://groups.google.com/forum/#!forum/wazuh
- Rule Documentation: https://documentation.wazuh.com/current/user-manual/ruleset/