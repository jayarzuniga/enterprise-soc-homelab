# Windows Client — Domain Join & Monitoring

## 1. Prerequisites & Network Configuration

### 1.1 What You Should Have Ready

Before proceeding, ensure you have:

- [ ] DC01 (Domain Controller) running and accessible
- [ ] Domain `lab.local` is operational
- [ ] DNS is working: `nslookup lab.local` returns DC01's IP
- [ ] Domain users created (`jdoe`, `jsmith`, etc.)
- [ ] Windows **Pro, Enterprise, or Education** edition installed *(NOT Home)*
- [ ] VirtualBox Internal Network configured

> ⚠️ **Windows Home edition CANNOT join Active Directory domains.** You must use Windows Pro, Enterprise, or Education.

> ⚠️ **My Lab Note:** The recommended topology uses separate subnets per segment. In my setup, all systems are on `192.168.3.x/24`. The IPs below reflect the best-practice layout — adjust to match your actual network.

---

### 1.2 VirtualBox Network Settings for Windows Client VMs

| Setting           | Value                                      |
| ----------------- | ------------------------------------------ |
| Adapter 1         | Internal Network                           |
| Name              | `Internal_Network` *(exact spelling)*      |
| Adapter Type      | Intel PRO/1000 MT Desktop (82540EM)        |
| Promiscuous Mode  | Allow All                                  |

---

### 1.3 Configure Static IP on Windows Client

**Step 1 — Open Network Settings:**

1. Right-click **Start** → **Network Connections**
2. Click **Change adapter options**
3. Right-click **Ethernet** → **Properties**
4. Double-click **Internet Protocol Version 4 (TCP/IPv4)**

**Step 2 — Configure IP Settings:**

| Setting           | Windows-Client-01      | Windows-Client-02      |
| ----------------- | ---------------------- | ---------------------- |
| IP address        | `192.168.10.10`        | `192.168.10.11`        |
| Subnet mask       | `255.255.255.0`        | `255.255.255.0`        |
| Default gateway   | `192.168.10.1`         | `192.168.10.1`         |
| Preferred DNS     | `192.168.20.10` (DC01) | `192.168.20.10` (DC01) |

Click **OK → OK → Close**

**Step 3 — Verify Network Connectivity:**

Open Command Prompt and run:

```cmd
ping 192.168.10.1       :: pfSense gateway — should succeed
ping 192.168.20.10      :: DC01 — should succeed
ping lab.local          :: should resolve to 192.168.20.10
nslookup lab.local      :: should return DC01's IP
```

> ⚠️ **If DNS resolution fails, you cannot join the domain.** Verify DC01 is running and the DNS service is started.

---

## 2. Join Windows Client to Active Directory Domain

### 2.1 Access System Properties

1. Right-click **Start** → **System**
2. Scroll down → Click **"Rename this PC (advanced)"**
3. In the *System Properties* window, click **Change...**

### 2.2 Join the Domain

1. Under *Member of*, select **Domain** *(not Workgroup)*
2. Type: `lab.local`
3. Click **OK**
4. Enter domain admin credentials when prompted:

   | Field    | Value                              |
   | -------- | ---------------------------------- |
   | Username | `LAB\Administrator` or `LAB\labadmin` |
   | Password | Your domain admin password         |

5. Click **OK**
6. Success message: *"Welcome to the lab domain!"* → Click **OK**
7. Click **Restart Now**

> ⚠️ The computer will restart automatically — this is required.

---

### 2.3 First Login to Domain

After restart:

| Field    | Client-01        | Client-02        |
| -------- | ---------------- | ---------------- |
| Username | `LAB\jdoe`       | `LAB\jsmith`     |
| Password | `ComplexPass123!`| `ComplexPass123!`|

> First login takes **2–3 minutes** while the user profile is created. This is normal.

> ✅ **Windows client is now domain-joined and authenticating via Active Directory.**

---

## 3. Verify Domain User Access

Open Command Prompt and run:

```cmd
whoami
```
Expected: `lab\jdoe`

```cmd
echo %USERDOMAIN%
```
Expected: `LAB`

---

## 4. Install Wazuh Agent on Windows Clients

Repeat these steps on **both** Windows client VMs.

**Step 1 — Download the installer:**

```
https://packages.wazuh.com/4.x/windows/wazuh-agent-4.7.5-1.msi
```

**Step 2 — Run the installer and configure:**

| Setting              | Client-01              | Client-02              |
| -------------------- | ---------------------- | ---------------------- |
| Wazuh server address | `192.168.3.81`         | `192.168.3.81`         |
| Port                 | `1514`                 | `1514`                 |
| Agent name           | `Windows-Client-01`    | `Windows-Client-02`    |

**Step 3 — Register and start the agent (PowerShell as Administrator):**

```powershell
cd "C:\Program Files (x86)\ossec-agent"
.\agent-auth.exe -m 192.168.3.81
NET START WazuhSvc
```

**Verify:**

```powershell
Get-Service WazuhSvc
Test-NetConnection 192.168.3.81 -Port 1514
```

---

## 5. Install & Configure Sysmon

**Step 1 — Download:**

- Sysmon: https://download.sysinternals.com/files/Sysmon.zip
- SwiftOnSecurity config: https://github.com/SwiftOnSecurity/sysmon-config

**Step 2 — Install:**

```powershell
# Extract Sysmon.zip to C:\Tools\Sysmon
# Save sysmonconfig-export.xml to C:\Tools\

cd C:\Tools\Sysmon
.\Sysmon64.exe -accepteula -i C:\Tools\sysmonconfig-export.xml
```

**Verify:**

```powershell
Get-Service Sysmon64
```

Expected: `Status = Running`

**Step 3 — Configure Wazuh to collect Sysmon events:**

```powershell
notepad "C:\Program Files (x86)\ossec-agent\ossec.conf"
```

Add inside the `<ossec_config>` block:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

**Restart Wazuh agent:**

```powershell
Restart-Service WazuhSvc
```

---

---

# Kali Linux — Attack Platform

## 7. Kali Linux Installation & Setup

### 7.1 Download Kali Linux

Download from the official Kali website:
https://www.kali.org/get-kali/#kali-installer-images

Select: **Installer Images → 64-bit** *(~4 GB)*

---

### 7.2 Create Kali VM in VirtualBox

| Setting        | Value                             |
| -------------- | --------------------------------- |
| Name           | `Kali-Linux`                      |
| Type           | Linux                             |
| Version        | Debian (64-bit)                   |
| RAM            | 4096 MB (4 GB)                    |
| CPU            | 2 cores                           |
| Disk           | 80 GB (dynamically allocated)     |
| Adapter 1      | Internal Network: `Attacker_Network` |

Attach the Kali Linux ISO to the optical drive, then start the VM.

---

### 7.3 Install Kali Linux

Follow the graphical installer:

| Step              | Value                                   |
| ----------------- | --------------------------------------- |
| Install type      | Graphical Install                       |
| Language          | English                                 |
| Hostname          | `kali`                                  |
| Domain name       | *(leave blank)*                         |
| Username          | `kali`                                  |
| Password          | `kali` *(or your choice)*               |
| Partitioning      | Guided — use entire disk, all files in one partition |
| GRUB              | Yes — install to primary drive          |

After installation completes, the VM will reboot automatically.

---

## 8. Network Configuration for Kali VM

After Kali boots, configure a static IP:

```bash
sudo nano /etc/network/interfaces
```

Add or modify:

```
auto eth0
iface eth0 inet static
    address 192.168.50.10
    netmask 255.255.255.0
    gateway 192.168.50.1
    dns-nameservers 8.8.8.8 8.8.4.4
```

Save (`Ctrl+X`, `Y`, `Enter`) and restart networking:

```bash
sudo systemctl restart networking
ip addr show eth0
```

> ⚠️ **My Lab Note:** In my single-subnet setup, Kali is at `192.168.3.30` with gateway `192.168.3.1`. Adjust the above to match your actual network.

---

## 9. Essential Tools Installation

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y bloodhound impacket-scripts crackmapexec responder
sudo apt install -y evil-winrm powershell-empire starkiller
```

**Tools overview:**

| Tool              | Purpose                                      |
| ----------------- | -------------------------------------------- |
| Nmap              | Network reconnaissance and port scanning     |
| BloodHound        | Active Directory attack path analysis        |
| Impacket          | Python scripts for SMB and Kerberos attacks  |
| CrackMapExec      | Post-exploitation and lateral movement       |
| Responder         | LLMNR/NBT-NS poisoning                       |
| Mimikatz          | Credential extraction *(run on Windows)*     |
| Evil-WinRM        | Windows Remote Management shell              |
| Metasploit        | Exploitation framework                       |

---

## 10. Configure Attack Tools

### 10.1 Download Mimikatz

> ⚠️ Mimikatz is blocked by Windows Defender — it must be transferred to the Windows target for testing, not run directly.

```bash
mkdir ~/tools && cd ~/tools
wget https://github.com/gentilkiwi/mimikatz/releases/download/2.2.0-20220919/mimikatz_trunk.zip
unzip mimikatz_trunk.zip
```

The `x64/mimikatz.exe` binary will be transferred to Windows client VMs when needed during attack simulations.

---

### 10.2 Configure Responder

```bash
sudo nano /usr/share/responder/Responder.conf
```

Verify these settings are enabled:

```
SMB = On
HTTP = On
```

Save and exit.

---

## 11. Test Connectivity & Initial Reconnaissance

**Basic connectivity test:**

```bash
ping 192.168.50.1      # pfSense gateway
ping 192.168.10.10     # Windows Client-01
ping 192.168.20.10     # Domain Controller
nmap -sn 192.168.10.0/24   # Discover hosts on Internal network
```

> ⚠️ Initial scans may be blocked by pfSense. You will configure firewall rules to allow controlled attack scenarios.

### 11.1 First Reconnaissance Test

```bash
nmap -sV -p 445,135,139 192.168.10.10
```

> 💡 This scan should **trigger a Suricata alert** on pfSense. Check Wazuh for the detection — this is your first end-to-end SOC test!

---

## 12. End-to-End Infrastructure Test

Run through this checklist to verify the complete lab is functional:

| Test               | Action                              | Expected Result                    |
| ------------------ | ----------------------------------- | ---------------------------------- |
| Domain Login       | Login as `LAB\jdoe`                 | Successful authentication          |
| DNS Resolution     | `nslookup dc01.lab.local`           | Returns `192.168.20.10`            |
| Wazuh Agents       | Check Wazuh Manager agent list      | All agents show **Active**         |
| Sysmon Logging     | Run `notepad.exe` on any client     | Event ID 1 appears in Wazuh        |
| Network Scan       | `nmap` from Kali                    | Suricata alert generated in Wazuh  |
| Lateral Access     | RDP from Client-01 to Client-02     | Logged as Event ID 4624 in Wazuh   |

---

## 13. Troubleshooting

### 13.1 Cannot Join Domain

| Symptom                          | Fix                                                      |
| -------------------------------- | -------------------------------------------------------- |
| DNS resolution fails             | Verify DNS points to `192.168.20.10`, check DC01 is running |
| "Domain not found"               | `ping 192.168.20.10` — verify network connectivity      |
| "Access denied" during join      | Use `LAB\Administrator` and verify password             |
| Join succeeds but login fails    | Ensure Windows Pro/Enterprise/Education edition          |

---

### 13.2 Wazuh Agent Not Connecting

```powershell
Get-Service WazuhSvc                                           # Service running?
Test-NetConnection 192.168.3.81 -Port 1514                     # Port reachable?
notepad "C:\Program Files (x86)\ossec-agent\ossec.conf"        # Correct manager IP?
```

Re-register if needed:

```powershell
cd "C:\Program Files (x86)\ossec-agent"
.\agent-auth.exe -m 192.168.3.81
Restart-Service WazuhSvc
```

---

### 13.3 Kali Cannot Reach Targets

```bash
ip addr show          # Verify IP is configured
ping 192.168.50.1     # Gateway reachable?
```

Also check: **pfSense → Firewall → Rules → Attacker_Network** — ensure outbound traffic rules exist. Test with ICMP (`ping`) before running complex attack tools.

---

## Conclusion

Your SOC homelab infrastructure is now complete. You have:

- A **Domain Controller** (DC01) running Active Directory with `lab.local`
- **Windows clients** domain-joined and monitored by Wazuh + Sysmon
- A **Wazuh SIEM** collecting events from all Windows and Linux systems
- A **pfSense firewall** with Suricata IDS/IPS forwarding alerts to Wazuh
- A **Kali Linux** attack platform ready for penetration testing and attack simulations

> 💡 Start with the **end-to-end test in Section 12** to confirm everything is wired together, then begin running attack scenarios and observing the detections in Wazuh.