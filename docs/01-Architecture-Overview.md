# SOC Homelab — Architecture Overview

## About This Lab

This homelab simulates an enterprise Security Operations Center (SOC) environment built entirely on a single physical machine using VirtualBox. It covers network segmentation, intrusion detection, Active Directory, centralized log management, and attack simulation.

> ⚠️ **Resource Note:** The recommended architecture uses fully isolated network segments (VLANs) per the ideal topology below. Due to hardware limitations, my actual deployment places all systems on a single `192.168.3.x/24` network. Both layouts are documented throughout the guides.

---

## Ideal Network Topology

```
Internet
   │
   ▼
ISP Router (192.168.0.1)
   │
   ▼
TP-Link Router
   │
   ▼
pfSense Firewall VM  ◄──── Suricata IDS/IPS (WAN + DMZ monitoring)
   │
   ├──── DMZ          192.168.3.0/24   Wazuh, NextCloud, Reverse Proxy
   ├──── INTERNAL     192.168.10.0/24  Windows Workstations
   ├──── SERVER       192.168.20.0/24  Domain Controller, Backend
   ├──── SOC          192.168.30.0/24  Monitoring Tools
   └──── ATTACKER     192.168.50.0/24  Kali Linux
```

---

## My Actual Lab Topology

```
Internet
   │
   ▼
ISP Router (192.168.0.1)
   │
   ▼
TP-Link Router
   │
   ▼
pfSense Firewall VM  ◄──── Suricata IDS/IPS
   │
   └──── DMZ_Network   192.168.3.0/24  (all VMs on single subnet)
              │
              ├── Wazuh Manager        192.168.3.81
              ├── Windows Server DC01  192.168.3.10
              ├── Windows Workstation  192.168.3.20
              └── Kali Linux           192.168.3.30
```

---

## System Inventory

| VM Name              | Role                          | OS                      | IP Address       | Network Segment     |
| -------------------- | ----------------------------- | ----------------------- | ---------------- | ------------------- |
| pfSense Firewall     | Firewall / Router / IDS/IPS   | FreeBSD (pfSense CE)    | 192.168.3.1      | Gateway             |
| Wazuh Manager        | SIEM / Log Aggregation        | Ubuntu 22.04 LTS        | 192.168.3.81     | DMZ (3.0/24)        |
| DC01                 | Active Directory / DNS        | Windows Server 2022     | 192.168.3.10     | DMZ (3.0/24)        |
| Windows Workstation  | Domain-joined client          | Windows Enterprise      | 192.168.3.20     | DMZ (3.0/24)        |
| Kali Linux           | Attack simulation platform    | Kali Linux              | 192.168.3.30     | DMZ (3.0/24)        |

---

## Component Roles

### 🔥 pfSense (Firewall + IDS/IPS)
- Acts as the perimeter firewall and router for all lab segments
- Runs **Suricata** for real-time intrusion detection and prevention
- Monitors both WAN and LAN interfaces with Emerging Threats rule sets
- Forwards all logs and Suricata alerts to Wazuh via syslog (UDP 514)
- Enforces inter-VLAN firewall rules between all network segments

### 🛡️ Wazuh Manager (SIEM)
- Central log aggregation and security event correlation engine
- Receives logs from all agents via **TCP 1514** (encrypted)
- Receives pfSense syslog and Suricata alerts via **UDP 514**
- Applies custom decoders and rules for pfSense and Suricata log formats
- Provides real-time alerting, file integrity monitoring, and vulnerability detection

### 🏢 DC01 — Windows Server 2022 (Active Directory)
- Hosts **Active Directory Domain Services** for domain `lab.local`
- Provides **DNS** for all domain-joined systems
- Contains simulated enterprise users, groups, and OUs for attack scenarios
- Monitored by **Wazuh Agent** + **Sysmon** (SwiftOnSecurity config)
- Advanced audit policies enabled to log authentication, account, and privilege events

### 💻 Windows Workstation (Windows Enterprise)
- Domain-joined workstation simulating an enterprise end-user machine
- Monitored by **Wazuh Agent** + **Sysmon**
- Used as a target for lateral movement, credential attacks, and RDP scenarios
- Logs all process creation, network connections, and file activity via Sysmon

### ☠️ Kali Linux (Attack Platform)
- Dedicated attacker VM for simulating real-world threat actor behavior
- Isolated on its own network segment (`Attacker_Network`) in the ideal topology
- Pre-loaded with: **BloodHound**, **Impacket**, **CrackMapExec**, **Responder**, **Evil-WinRM**, **Mimikatz**
- Used to generate attacks that Suricata and Wazuh are configured to detect

---

## Data & Log Flow

```
┌─────────────────────────────────────────────────────────┐
│                    ATTACK SIMULATION                    │
│  Kali Linux  →  nmap / Responder / Impacket / BloodHound│
└────────────────────────┬────────────────────────────────┘
                         │ Attack traffic
                         ▼
┌─────────────────────────────────────────────────────────┐
│               pfSense Firewall + Suricata               │
│  • Inspects all traffic on WAN and LAN interfaces       │
│  • Generates alerts for scans, exploits, malware        │
│  • Forwards alerts → Wazuh via syslog UDP:514           │
└────────────────────────┬────────────────────────────────┘
                         │ Syslog (UDP 514)
                         ▼
┌─────────────────────────────────────────────────────────┐
│                   Wazuh Manager (SIEM)                  │
│  • Receives pfSense + Suricata syslog                   │
│  • Receives Windows agent logs via TCP:1514             │
│  • Correlates events, applies rules, fires alerts       │
└────────┬───────────────┬──────────────────┬─────────────┘
         │ TCP:1514      │ TCP:1514          │ TCP:1514
         ▼               ▼                  ▼
  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐
  │ DC01 Agent  │  │  WS01 Agent  │  │  (future VMs)│
  │ + Sysmon    │  │  + Sysmon    │  │              │
  └─────────────┘  └──────────────┘  └──────────────┘
```

---

## Key Ports & Protocols

| Port      | Protocol | Direction              | Purpose                                  |
| --------- | -------- | ---------------------- | ---------------------------------------- |
| `514`     | UDP      | pfSense → Wazuh        | Syslog (pfSense + Suricata alerts)       |
| `1514`    | TCP      | Agents → Wazuh         | Wazuh agent–manager communication        |
| `1515`    | TCP      | Agents → Wazuh         | Wazuh agent enrollment/authentication    |
| `443`     | TCP      | Clients → pfSense      | pfSense web UI (HTTPS)                   |
| `53`      | UDP/TCP  | All VMs → DC01         | DNS resolution for `lab.local`           |
| `88`      | TCP/UDP  | Clients → DC01         | Kerberos authentication                  |
| `389`     | TCP      | Clients → DC01         | LDAP (Active Directory queries)          |
| `445`     | TCP      | Internal               | SMB (file sharing + lateral movement)    |
| `3389`    | TCP      | Internal               | RDP (monitored for lateral movement)     |

---

## Active Directory Structure

```
lab.local (Forest / Domain)
├── Builtin
├── Computers
├── Domain Controllers
│   └── DC01
├── LAB Admins
│   └── labadmin
├── LAB Computers
├── LAB Groups
│   ├── IT Admins
│   ├── IT Department
│   ├── HR Department
│   ├── Finance Department
│   └── Remote Desktop Users
├── LAB Servers
├── LAB Users
│   ├── jdoe        (John Doe — test user)
│   ├── jsmith      (Jane Smith — IT Dept)
│   ├── bjohnson    (Bob Johnson — HR Dept)
│   ├── awilliams   (Alice Williams — Finance)
│   └── svc_sql     (Service account — Kerberoasting target)
└── Users
```

---

## Security Monitoring Stack

| Layer               | Tool              | What It Detects                                      |
| ------------------- | ----------------- | ---------------------------------------------------- |
| Network perimeter   | Suricata (pfSense)| Port scans, exploits, malware C2, protocol anomalies |
| SIEM correlation    | Wazuh Manager     | Multi-source event correlation, custom rules         |
| Windows endpoint    | Sysmon            | Process creation, network connections, registry changes |
| Windows endpoint    | Wazuh Agent       | Authentication events, file integrity, audit logs    |
| AD authentication   | DC01 Audit Policy | Logon events, account changes, Kerberos activity     |

---

## Attack Scenarios This Lab Can Simulate

| Technique                | Tool Used          | Detection Layer              |
| ------------------------ | ------------------ | ---------------------------- |
| Network reconnaissance   | Nmap               | Suricata → Wazuh             |
| LLMNR/NBT-NS poisoning   | Responder          | Suricata + Wazuh             |
| Kerberoasting            | Impacket           | DC01 Audit + Wazuh Rule      |
| Pass-the-Hash            | CrackMapExec       | Wazuh (Event ID 4624/4625)   |
| BloodHound AD enumeration| BloodHound/SharpHound | Wazuh + Sysmon            |
| Credential dumping       | Mimikatz           | Sysmon Event ID 10 + Wazuh   |
| Lateral movement via RDP | Native RDP         | Wazuh (Event ID 4624)        |
| GPO abuse                | Manual / Empire    | DC01 Audit + Wazuh           |

---

## VirtualBox Network Adapters — pfSense VM

| Adapter | Attached to      | Internal Network Name | Purpose                    |
| ------- | ---------------- | --------------------- | -------------------------- |
| 1       | NAT              | —                     | WAN (internet access)      |
| 2       | Internal Network | `DMZ_Network`         | 192.168.3.0/24 — all VMs  |
| 3       | Internal Network | `Internal_Network`    | 192.168.10.0/24 (ideal)    |
| 4       | Internal Network | `Server_Network`      | 192.168.20.0/24 (ideal)    |
| 5       | Internal Network | `SOC_Network`         | 192.168.30.0/24 (ideal)    |
| 6       | Internal Network | `Attacker_Network`    | 192.168.50.0/24 (ideal)    |

---

## Lab Guides Index

| Guide                                    | Coverage                                              |
| ---------------------------------------- | ----------------------------------------------------- |
| `pfsense-suricata.md`                    | Introduction, network topology, objectives            |
| `prerequisites.md`                       | Software downloads and system requirements            |
| `virtualbox-pfsense-setup.md`            | VirtualBox network modes and pfSense VM creation      |
| `pfsense-installation.md`               | pfSense ISO installation and console configuration    |
| `pfsense-websetup-suricata-wazuh.md`     | Web UI setup, Suricata, firewall rules, Wazuh syslog  |
| `active-directory-setup-1-3.md`          | AD introduction, prerequisites, VM creation           |
| `active-directory-setup-4-6.md`          | Windows Server install, AD DS installation            |
| `active-directory-setup-7.md`            | OU structure and domain configuration                 |
| `active-directory-setup-8-9.md`          | Users, groups, and Group Policy                       |
| `active-directory-setup-10-11.md`        | Wazuh Agent and Sysmon on DC01                        |
| `active-directory-setup-12-14.md`        | Verification, hardening, and troubleshooting          |
| `wazuh-setup-guide.md`                   | Wazuh Manager install, all agent deployments          |
| `windows-client-kali-setup.md`           | Domain join, Windows client agents, Kali Linux setup  |