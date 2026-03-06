# 1. Introduction to pfSense and Suricata

## 1.1 What is pfSense?
pfSense is a free, open-source firewall and router distribution based on FreeBSD. It provides enterprise-grade network security featuring the following:
- Stateful **packet** filtering firewall
- Network Address Translation (NAT)
- Virtual Private Network (VPN) support (IPsec, OpenVPN, WireGuard)
- Traffic shaping and Quality of Service (QoS)
- Intrusion Detection/Prevention via Suricata and Snort
- Detailed logging and reporting
- Multi-WAN capabilities
- VLAN support for network segmentation

## 1.2 What is Suricata?
Suricata is a high-performance, open-source Network IDS/IPS and Network Security Monitoring engine. It provides:
- Real-time intrusion detection and prevention
- Protocol analysis and file extraction
- Multi-threaded packet processing for high performance
- Lua scripting for custom detection logic
- Compatible with Emerging Threats and Snort rules
- JSON logging for SIEM integration (perfect for Wazuh)
- TLS/SSL certificate logging
- DNS request and response logging
- HTTP transaction logging

## 1.3 Objectives for this SOC Homelab
- Simulate enterprise network architecture with VLANs and segmentation
- Practice firewall rule creation and management
- Detect network-based attacks in real-time with Suricata
- Monitor and analyze network traffic patterns
- Learn professional network security skills
- Integrate with Wazuh SIEM for centralized logging and correlation
- Test security controls against attack simulations

---

# 2. SOC Lab System Architecture

## 2.1 Network Topology

```
Internet → ISP Router (192.168.0.1) → TP-Link Router → pfSense Firewall (VM) → Segmented Networks
```

| Network  | Subnet          | Purpose                                         |
| -------- | --------------- | ----------------------------------------------- |
| WAN      | 192.168.0.x     | Internet connection via TP-Link router          |
| DMZ      | 192.168.3.0/24  | Public-facing servers (Wazuh, NextCloud, Proxy) |
| INTERNAL | 192.168.10.0/24 | User workstations and clients                   |
| SERVER   | 192.168.20.0/24 | Backend services (AD, databases, file servers)  |
| SOC      | 192.168.30.0/24 | Security monitoring tools (Zeek, dashboards)    |
| ATTACKER | 192.168.50.0/24 | Attack simulation (Kali Linux)                  |

> ⚠️ **My Lab Note:** The network plan above is best practice for a full SOC lab. However, due to limited resources, my setup uses different IP addresses — everything is placed in the `192.168.3.x/24` network. You may see IPs that don't match the plan above.

## 2.2 Detailed IP Address Plan
Below is the complete IP address allocation for all current and future systems in your SOC lab:

| System                            | IP address      |
| --------------------------------- | --------------- |
| pfSense DMZ Gateway               | 192.168.0.x     |
| Wazuh Manager (SIEM)              | 192.168.3.81/24 |
| Windows Server (Active Directory) | 192.168.3.10/24 |
| Windows  (Workstation)            | 192.168.3.20/24 |
| Kali Linux                        | 192.168.3.30/24 |

# 3. Prerequisites and Requirements

## 3.1 Required Software Downloads

### VirtualBox 7.x + Extension Pack
- **Download:** https://www.virtualbox.org/wiki/Downloads
- Free virtualization platform for running multiple operating systems.

### pfSense Community Edition ISO
- **Download:** https://www.pfsense.org/download/
- Select the following options:
  - **Architecture:** AMD64 (64-bit)
  - **Installer:** DVD Image (ISO)
  - **Mirror:** Choose closest mirror for fastest download

---

## 3.2 System Requirements

### pfSense VM Minimum Requirements

| Component          | Minimum                  | Recommended                          |
| ------------------ | ------------------------ | ------------------------------------ |
| CPU                | 2 cores (64-bit)         | 64-bit with AES-NI support           |
| RAM                | 2GB                      | 4GB (required for Suricata)          |
| Disk               | 20GB                     | 40GB                                 |
| Network Adapters   | 2 (WAN + LAN)            | Up to 6 for full segmentation        |
| OS                 | FreeBSD-based            | Included in pfSense ISO              |

---

## 3.3 Pre-Installation Checklist

- [ ] VirtualBox installed with Extension Pack added
- [ ] pfSense ISO downloaded and verified
- [ ] Basic networking knowledge (IP addresses, subnets, gateways)
- [ ] Existing network topology documented
- [ ] At least 30 minutes available for installation

> **Note:** This guide may differ based on the time it was created.

# 4. VirtualBox Network Configuration

Before creating the pfSense VM, we need to understand VirtualBox network modes and configure the appropriate adapters for our SOC lab topology.

## 4.1 Understanding VirtualBox Network Modes

| Mode             | Description                                              | Use Case in SOC Lab                        |
| ---------------- | -------------------------------------------------------- | ------------------------------------------ |
| NAT              | VM can access internet via host, not accessible outside  | pfSense WAN interface                      |
| Bridged          | VM appears as physical device on your network            | Alternative WAN (exposes VM to home network) |
| Internal Network | VMs on same internal network can communicate             | All pfSense LAN segments (DMZ, Internal, etc.) |

---

## 4.2 Internal Network Names

VirtualBox internal networks are created automatically when you assign them to VMs. We will use the following internal network names:

| Network Name       | Subnet           | Purpose                  |
| ------------------ | ---------------- | ------------------------ |
| `DMZ_Network`      | 192.168.3.0/24   | Public-facing servers    |
| `Internal_Network` | 192.168.10.0/24  | User workstations        |
| `Server_Network`   | 192.168.20.0/24  | Backend services         |
| `SOC_Network`      | 192.168.30.0/24  | Monitoring tools         |
| `Attacker_Network` | 192.168.50.0/24  | Attack simulation        |

> **Note:** These names are case-sensitive. Use them exactly as shown when configuring network adapters.

---

> ⚠️ **My Lab Note:** The network plan above is best practice for a full SOC lab. However, due to limited resources, my setup uses different IP addresses — everything is placed in the `192.168.3.x/24` network. You may see IPs that don't match the plan above.
>
> | System                            | IP Address       |
> | --------------------------------- | ---------------- |
> | pfSense DMZ Gateway               | 192.168.0.x      |
> | Wazuh Manager (SIEM)              | 192.168.3.81/24  |
> | Windows Server (Active Directory) | 192.168.3.10/24  |
> | Windows (Workstation)             | 192.168.3.20/24  |
> | Kali Linux                        | 192.168.3.30/24  |

---

# 5. Creating the pfSense Virtual Machine

## 5.1 Create New VM in VirtualBox

1. Open VirtualBox and click **New**
2. Configure basic settings:
   - **Name:** `pfSense-Firewall`
   - **Type:** BSD
   - **Version:** FreeBSD (64-bit)
3. Click **Next**
4. Set **Memory:** 4096 MB (4 GB)
5. Set **Processors:** 2 cores
6. Click **Next**
7. Configure **Hard Disk:**
   - Select *Create a virtual hard disk now*
   - **Size:** 40 GB
   - **Type:** VDI (VirtualBox Disk Image)
   - **Storage:** Dynamically allocated
8. Click **Finish**

---

## 5.2 Configure VM Settings

Select the `pfSense-Firewall` VM and click **Settings**.

### System Settings

1. Go to **System → Motherboard**
   - Boot Order: Ensure **Optical** is above **Hard Disk**
   - Uncheck **Floppy**
2. Go to **System → Processor**
   - Enable **PAE/NX** (if available)
   - Enable **VT-x/AMD-V** if supported

### Storage Settings

1. Go to **Storage**
2. Click on **Empty** under *Controller: IDE*
3. Click the disk icon → **Choose a disk file**
4. Select your pfSense ISO file

---

## 5.3 Configure Network Adapters

This is the most critical configuration step. We will set up **6 network adapters** for full SOC lab segmentation. Pay close attention to each adapter configuration.

### Adapter 1 — WAN Interface

| Setting           | Value                                    |
| ----------------- | ---------------------------------------- |
| Enable Adapter    | ✅ Checked                               |
| Attached to       | NAT                                      |
| Adapter Type      | Intel PRO/1000 MT Desktop (82540EM)      |
| Promiscuous Mode  | Allow All                                |
| Purpose           | Internet connection for pfSense          |

### Adapter 2 — DMZ

| Setting           | Value                                    |
| ----------------- | ---------------------------------------- |
| Enable Adapter    | ✅ Checked                               |
| Attached to       | Internal Network                         |
| Name              | `DMZ_Network`                            |
| Adapter Type      | Intel PRO/1000 MT Desktop (82540EM)      |
| Promiscuous Mode  | Allow All                                |
| Purpose           | 192.168.3.0/24 — Wazuh, NextCloud, Proxy |

### Adapter 3 — INTERNAL

| Setting           | Value                                    |
| ----------------- | ---------------------------------------- |
| Enable Adapter    | ✅ Checked                               |
| Attached to       | Internal Network                         |
| Name              | `Internal_Network`                       |
| Adapter Type      | Intel PRO/1000 MT Desktop (82540EM)      |
| Promiscuous Mode  | Allow All                                |
| Purpose           | 192.168.10.0/24 — User workstations      |

### Adapter 4 — SERVER

| Setting           | Value                                    |
| ----------------- | ---------------------------------------- |
| Enable Adapter    | ✅ Checked                               |
| Attached to       | Internal Network                         |
| Name              | `Server_Network`                         |
| Adapter Type      | Intel PRO/1000 MT Desktop (82540EM)      |
| Promiscuous Mode  | Allow All                                |
| Purpose           | 192.168.20.0/24 — AD, databases          |

### Adapter 5 — SOC

| Setting           | Value                                    |
| ----------------- | ---------------------------------------- |
| Enable Adapter    | ✅ Checked                               |
| Attached to       | Internal Network                         |
| Name              | `SOC_Network`                            |
| Adapter Type      | Intel PRO/1000 MT Desktop (82540EM)      |
| Promiscuous Mode  | Allow All                                |
| Purpose           | 192.168.30.0/24 — Zeek, monitoring       |

### Adapter 6 — ATTACKER

| Setting           | Value                                    |
| ----------------- | ---------------------------------------- |
| Enable Adapter    | ✅ Checked                               |
| Attached to       | Internal Network                         |
| Name              | `Attacker_Network`                       |
| Adapter Type      | Intel PRO/1000 MT Desktop (82540EM)      |
| Promiscuous Mode  | Allow All                                |
| Purpose           | 192.168.50.0/24 — Kali Linux             |

Click **OK** to save all settings.


# 6. Installing pfSense

## 6.1 Boot from ISO and Start Installation

1. Select the `pfSense-Firewall` VM and click **Start**
2. pfSense will boot from the ISO and display a copyright notice
3. Accept the copyright by pressing **Enter**
4. When the installer menu appears, select **Install**
5. Press **Enter** to continue

---

## 6.2 Partition the Disk

1. **Keymap Selection:** Select your keyboard layout (default is US)
2. **Partitioning:** Select **Auto (UFS)** for automatic partitioning
3. **Disk Selection:** Select the virtual disk (`ada0`)
4. **Partition Scheme:** Select **GPT** (GUID Partition Table)
5. **Confirm:** Select **Yes** to proceed with installation

> ⏳ Installation will take 2–5 minutes. A progress bar will show file extraction. **Do not close the VM window during installation.**

---

## 6.3 Complete Installation

1. When prompted **"Manual Configuration"**, select **No**
2. Select **Reboot**
3. When the VM reboots, remove the ISO:
   - VirtualBox menu: **Devices → Optical Drives → Remove disk**
4. pfSense will boot from the hard disk and display the console menu

---

# 7. Initial pfSense Configuration

## 7.1 Assign Interfaces

After pfSense boots, you will see the console menu. We need to assign network interfaces to match our VirtualBox adapter configuration.

1. **Should VLANs be set up now?** Type `n` and press **Enter**
2. **Enter the WAN interface name:** `em0` *(VirtualBox Adapter 1 — NAT)*
3. **Enter the LAN interface name:** `em1` *(VirtualBox Adapter 2 — DMZ)*
4. **Enter optional interfaces:**

   | Interface | Name  | Network          |
   | --------- | ----- | ---------------- |
   | OPT1      | `em2` | Internal Network |
   | OPT2      | `em3` | Server Network   |
   | OPT3      | `em4` | SOC Monitoring   |
   | OPT4      | `em5` | Attacker Lab     |

5. Press **Enter** when done adding optional interfaces
6. **Proceed with assignment?** Type `y` and press **Enter**

> ♻️ pfSense will apply the configuration and reboot. Wait for the console menu to appear again.

---

## 7.2 Configure LAN Interface IP Address

Set the DMZ (LAN) interface IP address to `192.168.3.1` to match your network plan.

1. From the console menu, select **option 2** — *Set interface(s) IP address*
2. **Select interface:** `2` *(LAN — em1 — DMZ)*
3. **Configure IPv4 address via DHCP?** Type `n`
4. **Enter new LAN IPv4 address:** `192.168.3.1`
5. **Enter subnet bit count:** `24`
6. **For upstream gateway**, press **Enter** *(none)*
7. **Configure IPv6 address via DHCP6?** Type `n`
8. **For IPv6 address**, press **Enter** *(none)*
9. **Enable DHCP server on LAN?** Type `y`
10. **Start address of DHCP range:** `192.168.3.100`
11. **End address of DHCP range:** `192.168.3.200`
12. **Revert to HTTP as webConfigurator protocol?** Type `n`

> ✅ Configuration complete! The web interface will be available at: **https://192.168.3.1**

# 8. pfSense Web Interface Setup

## 8.1 Access Web Interface

To access the pfSense web interface, you need a VM on the same DMZ network. You have two options:

### Option 1: Use Existing VM *(Recommended)*

Configure one of your existing VMs (Wazuh, NextCloud, or Reverse Proxy) to temporarily connect to `DMZ_Network`:

1. Shut down an existing VM (e.g., Wazuh Manager)
2. Go to **VM Settings → Network → Adapter 1**
   - **Attached to:** Internal Network
   - **Name:** `DMZ_Network`
3. Start the VM
4. Open a web browser and navigate to: **https://192.168.3.1**
5. Accept the security warning (self-signed certificate)

### Option 2: Create Temporary Management VM

Create a lightweight Ubuntu Desktop VM (1GB RAM, `DMZ_Network` adapter) specifically for pfSense management.

---

## 8.2 Login to pfSense

1. Navigate to: **https://192.168.3.1**
2. Enter default credentials:

   | Field    | Value      |
   | -------- | ---------- |
   | Username | `admin`    |
   | Password | `pfsense`  |

3. Click **Sign In**

> ⚠️ **WARNING:** Change the default password immediately after login for security!

---

## 8.3 Complete Setup Wizard

The setup wizard will guide you through initial configuration:

| Step | Section               | Action                                                                 |
| ---- | --------------------- | ---------------------------------------------------------------------- |
| 1    | Welcome               | Click **Next**                                                         |
| 2    | Netgate Support       | Click **Next** *(not required for homelab)*                            |
| 3    | General Information   | Hostname: `pfsense`, Domain: `localdomain`, DNS: `8.8.8.8` / `8.8.4.4` |
| 4    | Time Server           | Server: `pool.ntp.org`, select your timezone                           |
| 5    | Configure WAN         | Type: **DHCP**, uncheck *Block bogon networks* (for lab use)           |
| 6    | Configure LAN         | IP: `192.168.3.1`, Subnet Mask: `24`                                   |
| 7    | Set Admin Password    | Enter and confirm a strong password                                    |
| 8    | Reload                | Click **Reload** to apply changes                                      |
| 9    | Complete              | Click **Finish**                                                       |

> ✅ You will be redirected to the pfSense Dashboard!

---

# 9. Configure OPT Interface IP Addresses

After the wizard, the additional interfaces (OPT1–OPT4) need IP addresses assigned. Repeat the steps below for each interface.

Navigate to **Interfaces → [OPT interface name]** and configure:

| Interface | Assignment | IP Address       | Description      |
| --------- | ---------- | ---------------- | ---------------- |
| OPT1      | em2        | 192.168.10.1/24  | INTERNAL         |
| OPT2      | em3        | 192.168.20.1/24  | SERVER           |
| OPT3      | em4        | 192.168.30.1/24  | SOC              |
| OPT4      | em5        | 192.168.50.1/24  | ATTACKER         |

For each interface:

1. Go to **Interfaces → OPT[x]**
2. Check **Enable Interface**
3. Set **IPv4 Configuration Type:** Static IPv4
4. Enter the **IPv4 Address** and subnet `/24`
5. Set a meaningful **Description** (e.g., `INTERNAL`, `SERVER`, `SOC`, `ATTACKER`)
6. Click **Save**, then click **Apply Changes**

> ⚠️ **My Lab Note:** Due to limited resources, all VMs are placed on the `192.168.3.x/24` network. If you are following my setup, skip OPT interface configuration and use only the DMZ (LAN) interface.

---

# 10. Configure DHCP for OPT Interfaces

Enable DHCP on each OPT interface so VMs on those segments get IP addresses automatically.

Navigate to **Services → DHCP Server** and select each interface tab:

| Interface | DHCP Range Start | DHCP Range End  |
| --------- | ---------------- | --------------- |
| INTERNAL  | 192.168.10.100   | 192.168.10.200  |
| SERVER    | 192.168.20.100   | 192.168.20.200  |
| SOC       | 192.168.30.100   | 192.168.30.200  |
| ATTACKER  | 192.168.50.100   | 192.168.50.200  |

For each tab:

1. Check **Enable DHCP server on [interface]**
2. Enter the **Range** (start and end addresses)
3. Click **Save**

> ⚠️ **My Lab Note:** Skip this section if using a single `192.168.3.x/24` network — DHCP was already configured during console setup.

---

# 11. Configure Firewall Rules

Firewall rules control traffic between network segments. pfSense applies rules per interface — traffic is evaluated as it **enters** an interface.

## 11.1 Basic LAN/DMZ Rules

Navigate to **Firewall → Rules → LAN (DMZ)**:

1. Click **Add** (↑ to add at top)
2. Configure a rule to allow all DMZ traffic outbound:

   | Setting          | Value             |
   | ---------------- | ----------------- |
   | Action           | Pass              |
   | Interface        | LAN               |
   | Protocol         | Any               |
   | Source           | LAN subnets       |
   | Destination      | Any               |
   | Description      | Allow DMZ out     |

3. Click **Save**, then **Apply Changes**

## 11.2 Inter-VLAN Rules *(Skip if using single network)*

To allow or deny traffic between segments, add rules on each OPT interface. Best practice for a SOC lab:

| Rule                          | Source           | Destination      | Action |
| ----------------------------- | ---------------- | ---------------- | ------ |
| Allow INTERNAL to internet    | 192.168.10.0/24  | Any              | Pass   |
| Allow SERVER from INTERNAL    | 192.168.10.0/24  | 192.168.20.0/24  | Pass   |
| Allow SOC to all segments     | 192.168.30.0/24  | Any              | Pass   |
| Block ATTACKER to SOC/SERVER  | 192.168.50.0/24  | 192.168.20.0/24  | Block  |
| Allow ATTACKER to INTERNAL    | 192.168.50.0/24  | 192.168.10.0/24  | Pass   |

Navigate to **Firewall → Rules → [OPT interface]**, add rules in order, then click **Apply Changes**.

> 💡 Rules are evaluated top-to-bottom. Place more specific rules above general ones.

---

# 12. Installing Suricata IDS/IPS

Suricata provides intrusion detection and prevention capabilities for your SOC lab. It will analyze network traffic and generate alerts for suspicious activity.

## 12.1 Install Suricata Package

1. Navigate to **System → Package Manager**
2. Click the **Available Packages** tab
3. Search for: `suricata`
4. Click **Install** next to the Suricata package
5. Click **Confirm** to confirm installation
6. Wait for installation to complete *(2–5 minutes)*

---

## 12.2 Configure Global Settings

1. Navigate to **Services → Suricata**
2. Click the **Global Settings** tab
3. Configure:

   | Setting                         | Value                   |
   | ------------------------------- | ----------------------- |
   | Enable ET Open                  | ✅ Checked              |
   | Update Interval                 | 12 hours                |
   | Remove Blocked Hosts Interval   | 1 hour                  |

4. Click **Save**

> **Note:** Snort rules require a free registered account at snort.org. ET Open rules are sufficient for a SOC lab and require no registration.

---

## 12.3 Download Rule Sets

1. Click the **Updates** tab
2. Click **Update Rules**
3. Wait for download to complete *(5–10 minutes depending on connection)*

---

## 12.4 Create Suricata Interface for WAN

1. Click the **Interfaces** tab
2. Click **Add**
3. Configure:

   | Setting                      | Value                                                      |
   | ---------------------------- | ---------------------------------------------------------- |
   | Enable                       | ✅ Checked                                                 |
   | Interface                    | WAN (em0)                                                  |
   | Description                  | WAN Monitoring                                             |
   | Send Alerts to System Log    | ✅ Checked ← *Critical: sends alerts to Wazuh*            |
   | Block Offenders              | ✅ Checked *(IPS mode)*                                    |

4. Click **Save**

---

## 12.5 Create Suricata Interface for DMZ/LAN

Repeat the same process to monitor traffic on your internal DMZ segment:

1. Click **Add** again on the **Interfaces** tab
2. Configure:

   | Setting                      | Value                      |
   | ---------------------------- | -------------------------- |
   | Enable                       | ✅ Checked                 |
   | Interface                    | LAN (em1)                  |
   | Description                  | DMZ Monitoring             |
   | Send Alerts to System Log    | ✅ Checked                 |
   | Block Offenders              | ✅ Checked                 |

3. Click **Save**

> 💡 Monitoring both WAN and LAN gives you visibility into external threats **and** internal lateral movement — essential for a SOC lab.

---

# 13. Enable Suricata on Interfaces

After creating interfaces, start Suricata on each one:

1. Navigate to **Services → Suricata → Interfaces**
2. For each interface (WAN, LAN), click the **▶ Start** button under the *Status* column
3. Verify the status shows a **green play icon** indicating Suricata is running

> ⏳ Suricata may take 30–60 seconds to initialize on first start.

---

# 14. Enable Suricata Rule Categories

Enable relevant detection categories so Suricata actively monitors for threats:

1. Navigate to **Services → Suricata → Interfaces → WAN → Rules**
2. Enable recommended categories:

   | Category                        | Purpose                              |
   | ------------------------------- | ------------------------------------ |
   | `emerging-malware`              | Malware communications               |
   | `emerging-attack_response`      | Attack response detection            |
   | `emerging-exploit`              | Exploit attempts                     |
   | `emerging-scan`                 | Port scans and reconnaissance        |
   | `emerging-trojan`               | Trojan activity                      |
   | `emerging-dos`                  | Denial of service attacks            |
   | `emerging-policy`               | Policy violations                    |

3. Click **Save** after enabling categories
4. Repeat for the **LAN (DMZ)** interface

---

# 15. Integration with Wazuh SIEM

## 15.1 Configure Syslog in pfSense

1. Navigate to **Status → System Logs → Settings**
2. Scroll to **Remote Logging Options**
3. Configure:

   | Setting                  | Value                              |
   | ------------------------ | ---------------------------------- |
   | Enable Remote Logging    | ✅ Checked                         |
   | Remote Log Servers       | `192.168.3.81:514`                 |
   | Remote Syslog Contents   | Everything                         |

4. Click **Save**

---

## 15.2 Configure Wazuh to Receive Syslog

The `<remote>` syslog block already exists in `ossec.conf` by default. Verify it is present and the allowed IP matches pfSense:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Confirm this block exists:

```xml
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>udp</protocol>
  <allowed-ips>192.168.3.1/24</allowed-ips>
</remote>
```

> **Note:** Use `/24` subnet instead of a single IP so all pfSense interfaces are covered.

---

## 15.3 Create Custom Decoder for Suricata Syslog

Suricata logs from pfSense arrive without a standard hostname prefix — `suricata[PID]:` becomes the hostname in Wazuh's pre-decoder. A custom decoder is required.

```bash
sudo nano /var/ossec/etc/decoders/local_decoder.xml
```

Add these decoders:

```xml
<decoder name="pfsense">
  <program_name>^pfsense</program_name>
</decoder>

<decoder name="pfsense-firewall">
  <parent>pfsense</parent>
  <prematch>filterlog:</prematch>
  <regex offset="after_prematch">
    ^(\S+),(\S+),(\S+),(\S+),(\S+),(\S+),(\S+),(\S+),(\S+),(\S+),(\S+),(\S+),(\S+),(\S+),(\d+),(\S+),(\S+),(\d+),(\S+)
  </regex>
  <order>
    timestamp,rule_number,sub_rule_number,anchor,tracker,interface,reason,action,direction,
    ip_version,tos,ecn,ttl,id,offset,flags,proto_id,proto,length,src_ip,dst_ip,src_port,
    dst_port,data_length,tcp_flags,sequence_number
  </order>
</decoder>

<!-- Suricata logs from pfSense via syslog - no hostname prefix -->
<decoder name="suricata-syslog">
  <prematch>^suricata[</prematch>
</decoder>
```

> **Key lessons learned:**
> - Do **NOT** use `\d` in Wazuh decoders — it is not supported in the OSRegex engine
> - Do **NOT** use `<program_name>` for pfSense Suricata logs — the PID bracket causes `suricata[PID]:` to be parsed as hostname, not program name
> - The `pfsense-firewall` regex must **NOT** end with `,>` — the `>` breaks XML parsing and crashes `wazuh-analysisd`

---

## 15.4 Create Custom Rules for Suricata

```bash
sudo nano /var/ossec/etc/rules/local_rules.xml
```

```xml
<group name="suricata,ids,">
  <rule id="200100" level="5">
    <decoded_as>suricata-syslog</decoded_as>
    <description>Suricata alert detected (syslog)</description>
  </rule>

  <rule id="200101" level="12">
    <decoded_as>json</decoded_as>
    <field name="event_type">alert</field>
    <field name="alert.severity" type="pcre2">^[1-2]$</field>
    <description>Suricata: High severity alert</description>
    <mitre>
      <id>T1055</id>
    </mitre>
  </rule>

  <rule id="200102" level="8">
    <decoded_as>json</decoded_as>
    <field name="event_type">alert</field>
    <field name="alert.category">Trojan</field>
    <description>Suricata: Trojan detection</description>
  </rule>

  <rule id="200103" level="10">
    <decoded_as>json</decoded_as>
    <field name="event_type">alert</field>
    <field name="alert.signature">ET MALWARE</field>
    <description>Suricata: Malware signature detected</description>
  </rule>
</group>
```

> **Note:** Rule `200100` must **NOT** have `<field name="full_log">SURICATA</field>` — this prevents the rule from firing. The `<decoded_as>` match alone is sufficient.

---

## 15.5 Validate and Restart

```bash
sudo /var/ossec/bin/wazuh-analysisd -t
sudo systemctl restart wazuh-manager
```

---

## 15.6 Verify Integration

Test the decoder and rule with `wazuh-logtest`:

```bash
sudo /var/ossec/bin/wazuh-logtest
```

Paste a sample Suricata log:

```
Feb 27 19:24:50 suricata[14478]: [1:2200075:2] SURICATA UDPv4 invalid checksum [Classification: Generic Protocol Command Decode] [Priority: 3] {UDP} 10.0.2.15:8257 -> 8.8.8.8:53
```

Expected output:

```
Phase 2: name: 'suricata-syslog'
Phase 3: id: '200100', level: '5', description: 'Suricata alert detected (syslog)'
**Alert to be generated.
```

Check live alerts:

```bash
sudo grep -a "SURICATA" /var/ossec/logs/alerts/alerts.json | grep -v "sudo" | tail -10
```

---

## 15.7 View in Wazuh Dashboard

1. Open **Wazuh Dashboard → Discover**
2. Select index: `wazuh-alerts-*`
3. Set time range: **Last 24 hours**
4. Search:

```
rule.id: 200100
```

or

```
rule.groups: suricata
```

> ✅ You should see Suricata alerts from pfSense with `location: 192.168.3.1` appearing in real time.

---

# 16. Troubleshooting Common Issues

## 16.1 Cannot Access pfSense Web Interface

**Problem:** Browser cannot reach `https://192.168.3.1`

**Solutions:**
- Verify your VM is on `DMZ_Network` internal network
- Check VM IP address: `ip addr show` (should be `192.168.3.x`)
- Ping pfSense: `ping 192.168.3.1`
- Verify pfSense is running (check VirtualBox Manager)
- Try a different browser or incognito mode

---

## 16.2 Suricata Not Blocking Traffic

**Problem:** Suricata generates alerts but does not block malicious traffic

**Solutions:**
- Verify **Block Offenders** is checked in Suricata interface settings
- Confirm IPS mode is enabled (not just IDS)
- Ensure rules are enabled: **Services → Suricata → WAN → Rules**
- Check that active categories are enabled
- Restart Suricata: **Services → Suricata → Restart**

---

## 16.3 Wazuh Not Receiving pfSense Logs

**Problem:** pfSense logs not appearing in Wazuh

**Solutions:**
- Verify syslog is configured in pfSense: **Status → System Logs → Settings**
- Allow UDP port 514 on Wazuh: `sudo ufw allow 514/udp`
- Verify `ossec.conf` has the correct remote syslog block
- Check Wazuh is listening: `sudo netstat -ulnp | grep 514`
- Test connectivity from pfSense: `ping 192.168.3.81`
- Restart Wazuh: `sudo systemctl restart wazuh-manager`

---

## 16.4 OPT Interfaces Not Passing Traffic

**Problem:** VMs on OPT networks cannot communicate or reach the internet

**Solutions:**
- Verify the interface is enabled: **Interfaces → OPT[x] → Enable Interface**
- Confirm firewall rules exist for the interface: **Firewall → Rules → OPT[x]**
- Check DHCP server is enabled: **Services → DHCP Server → [interface]**
- Verify the VM's network adapter is set to the correct Internal Network name (case-sensitive)

---

# 17. Maintenance and Updates

## 17.1 Update pfSense

Regular updates are critical for security. Update pfSense monthly:

1. Navigate to **System → Update**
2. Click **Check for Updates**
3. If an update is available, review the release notes
4. **Create a VM snapshot before updating:** VirtualBox → Snapshots → Take
5. Click **Confirm** to install the update
6. Wait for update to complete *(5–10 minutes)*
7. System will reboot automatically

---

## 17.2 Update Suricata Rules

Keep detection rules current for the latest threat signatures:

1. Navigate to **Services → Suricata → Updates**
2. Click **Update Rules**
3. Wait for download to complete
4. Rules are automatically applied to all interfaces

> 💡 **Recommendation:** Enable automatic rule updates in Global Settings with a 12-hour update interval.

---

## 17.3 Monitor System Health

Regularly check pfSense system health using these dashboard sections:

| Location                          | What to Check                              |
| --------------------------------- | ------------------------------------------ |
| Status → Dashboard                | Overview of CPU, RAM, and network usage    |
| Status → Interfaces               | Verify all interfaces are UP               |
| Status → Services                 | All services should show as running        |
| Status → System Logs              | Review for errors or warnings              |
| Services → Suricata → Alerts      | Review security alerts regularly           |

---

## 17.4 Backup Configuration

Backup your pfSense configuration regularly:

1. Navigate to **Diagnostics → Backup & Restore**
2. Click **Download configuration as XML**
3. Save the file to a secure location
4. Create VirtualBox snapshots before any major changes

> 💡 **Recommendation:** Perform backups at minimum once per month.