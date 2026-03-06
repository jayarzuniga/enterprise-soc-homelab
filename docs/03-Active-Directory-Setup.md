# 1. Introduction & Overview

## 1.1 What is Active Directory?

Active Directory (AD) is Microsoft's directory service used by over 90% of enterprises worldwide. It provides centralized authentication, authorization, and management for Windows networks.

**Key features:**
- Centralized user and computer management
- Single sign-on (SSO) across the network
- Group Policy for configuration management
- Kerberos-based authentication
- LDAP directory services
- Certificate services integration

---

## 1.2 Why Active Directory in Your SOC Lab?

Active Directory is the **primary target** in enterprise attacks. By deploying AD in your SOC lab, you can practice detecting and responding to:

| Attack Technique              | Description                                          |
| ----------------------------- | ---------------------------------------------------- |
| **Kerberoasting**             | Extracting service account credentials               |
| **Pass-the-Hash**             | Using NTLM hashes for lateral movement               |
| **Golden Ticket**             | Forging Kerberos TGTs for domain persistence         |
| **DCSync**                    | Replicating domain credentials                       |
| **GPO Abuse**                 | Malicious Group Policy Objects                       |
| **Credential Dumping**        | Using Mimikatz to extract credentials from memory    |
| **LLMNR/NBT-NS Poisoning**    | Intercepting name resolution to capture hashes       |
| **BloodHound Analysis**       | Mapping attack paths through AD relationships        |

---

## 1.3 Lab Architecture

This Domain Controller will serve as the authentication hub for all Windows systems in your lab:

| Component               | IP Address      | Role                                  |
| ----------------------- | --------------- | ------------------------------------- |
| Domain Controller (DC01)| 192.168.20.10   | AD DS, DNS, Authentication            |
| Windows Client 01       | 192.168.10.10   | Domain-joined workstation             |
| Windows Client 02       | 192.168.10.11   | Domain-joined workstation             |
| Wazuh Manager           | 192.168.3.81    | Receives logs from all Windows systems|

> ⚠️ **My Lab Note:** Due to limited resources, all systems in my setup are on the `192.168.3.x/24` network. The IPs above reflect the recommended best-practice topology — yours may differ.

---

# 2. Prerequisites & Downloads

## 2.1 Download Windows Server 2022 ISO

**Official Microsoft Evaluation Center:**
https://www.microsoft.com/en-us/evalcenter/download-windows-server-2022

| Setting           | Value                                          |
| ----------------- | ---------------------------------------------- |
| Edition           | Windows Server 2022 Standard (Desktop Experience) |
| Format            | ISO *(not VHD)*                                |
| File Size         | ~5.2 GB                                        |
| Evaluation Period | 180 days *(extendable)*                        |
| Language          | English (United States)                        |

> **Note:** You will need to provide an email address — a Microsoft account is not required.

---

## 2.2 Alternative Download Methods

- **Option 1 — Windows Insider Program** *(Requires Microsoft Account)*: Access preview builds and extended trial periods.
- **Option 2 — MSDN / Visual Studio Subscription**: Full licensed versions if you have an active subscription.
- **Option 3 — Direct Download Links** *(May Expire)*: If the Evaluation Center link doesn't work, search "Windows Server 2022 download" on Microsoft's website. The evaluation version is always free for testing.

---

## 2.3 System Requirements

### Virtual Machine Specifications

| Component         | Minimum              | Recommended                        |
| ----------------- | -------------------- | ---------------------------------- |
| CPU               | 2 cores              | 4 cores                            |
| RAM               | 4 GB                 | 8 GB                               |
| Disk Space        | 60 GB (dynamic)      | 60 GB                              |
| Network Adapter 1 | Internal Network     | `Server_Network`                   |
| Network Adapter 2 | NAT                  | Temporary — for Windows Updates only |
| DVD Drive         | Windows Server 2022 ISO attached | —                      |

---

## 2.4 Additional Downloads *(After Server Installation)*

You will download these later in the guide:

| Tool                          | Source                    |
| ----------------------------- | ------------------------- |
| Wazuh Agent (4.7.5)           | packages.wazuh.com        |
| Sysmon (Latest)               | Microsoft Sysinternals    |
| SwiftOnSecurity Sysmon Config | GitHub                    |

---

# 3. VirtualBox VM Creation

## 3.1 Create New Virtual Machine

**Step 1 — Basic Settings:**

1. Open VirtualBox and click **New**
2. Configure:

   | Field   | Value                    |
   | ------- | ------------------------ |
   | Name    | `DC01`                   |
   | Folder  | Default or your choice   |
   | Type    | Microsoft Windows        |
   | Version | Windows 2022 (64-bit)    |

3. Click **Next**

**Step 2 — Memory:**

1. Set memory to **4096 MB** minimum *(8192 MB recommended if host has 16+ GB RAM)*
2. Click **Next**

**Step 3 — Processors:**

1. Set **Processors:** 2 CPUs *(4 recommended if available)*
2. Enable **PAE/NX:** ✅ Checked
3. Click **Next**

**Step 4 — Virtual Hard Disk:**

| Setting              | Value                         |
| -------------------- | ----------------------------- |
| Action               | Create a virtual hard disk now |
| Hard disk file type  | VDI (VirtualBox Disk Image)   |
| Storage              | Dynamically allocated         |
| File size            | 60 GB                         |

Click **Create**

---

## 3.2 Configure VM Settings

**Step 1:** Select the `DC01` VM → Click **Settings**

**Step 2 — System Settings:**

1. Go to **System → Motherboard**
   - Boot Order: Move **Optical** above **Hard Disk**
   - Uncheck **Floppy**
2. Go to **System → Processor**
   - Enable **PAE/NX** (if available)

**Step 3 — Storage Settings:**

1. Go to **Storage**
2. Click **Empty** under *Controller: IDE*
3. Click the disk icon → **Choose a disk file**
4. Select your **Windows Server 2022 ISO**
5. Verify it appears in the storage list

---

## 3.3 Configure Network Adapters

> ⚠️ **CRITICAL:** Network configuration is essential for proper lab operation. Double-check each setting carefully.

### Adapter 1 — Server Network *(Primary)*

| Setting           | Value                               |
| ----------------- | ----------------------------------- |
| Enable Adapter    | ✅ Checked                          |
| Attached to       | Internal Network                    |
| Name              | `Server_Network` *(case-sensitive)* |
| Adapter Type      | Intel PRO/1000 MT Desktop (82540EM) |
| Promiscuous Mode  | Allow All                           |
| Cable Connected   | ✅ Checked                          |

> **Why `Server_Network`?** This connects DC01 to the SOC lab's server segment (`192.168.20.0/24`) where it communicates with other backend services through pfSense routing.

### Adapter 2 — NAT *(Temporary, for Windows Updates)*

| Setting           | Value                               |
| ----------------- | ----------------------------------- |
| Enable Adapter    | ✅ Checked                          |
| Attached to       | NAT                                 |
| Adapter Type      | Intel PRO/1000 MT Desktop (82540EM) |
| Cable Connected   | ✅ Checked                          |

> **Note:** You will **disable Adapter 2** after initial setup and Windows Updates are complete. In production, a Domain Controller should only have internal network connectivity.

---

### Pre-Flight Checklist

Before continuing, verify all settings are correct:

- [ ] Windows Server 2022 ISO is attached
- [ ] 4–8 GB RAM allocated
- [ ] 2–4 CPU cores assigned
- [ ] 60 GB disk created
- [ ] Adapter 1 = Internal Network `Server_Network`
- [ ] Adapter 2 = NAT

Click **OK** to save settings.

# 4. Windows Server Installation

## 4.1 Boot from ISO

1. Select the `DC01` VM in VirtualBox and click **Start**
2. The VM will boot from the Windows Server ISO
3. Press any key when prompted: *"Press any key to boot from CD or DVD..."*
4. Wait for Windows Setup to load *(1–2 minutes)*

---

## 4.2 Windows Setup Wizard

**Screen 1 — Language and Region:**

| Setting                    | Value                  |
| -------------------------- | ---------------------- |
| Language to install        | English (United States)|
| Time and currency format   | English (United States)|
| Keyboard or input method   | US                     |

Click **Next**

**Screen 2 — Install Now:**
Click **"Install now"**

**Screen 3 — Product Key:**
Select **"I don't have a product key"** *(Evaluation version activates automatically)*

**Screen 4 — Select Operating System:**

> ⚠️ **CRITICAL:** Choose the correct edition!

✅ **Select:** `Windows Server 2022 Standard Evaluation (Desktop Experience)`

❌ **Do NOT select:**
- `Datacenter Evaluation` — More features than needed for a lab
- `Standard Evaluation` *(without "Desktop Experience")* — No GUI, command-line only

Click **Next**

**Screen 5 — License Terms:**
- Check **"I accept the Microsoft Software License Terms"**
- Click **Next**

**Screen 6 — Installation Type:**
- Select **"Custom: Install Microsoft Server Operating System only (advanced)"**
- Do **NOT** select Upgrade — this is a fresh installation

**Screen 7 — Disk Partitioning:**
- You should see *"Drive 0 Unallocated Space"* (60 GB)
- Click **Next** — Windows will create partitions automatically

> ⏳ Installation begins and takes **10–20 minutes**. Progress will show: *Copying files → Installing features → Installing updates → Finishing up*. The VM will reboot automatically.

---

## 4.3 Initial Setup (After Installation)

**Screen 8 — Set Administrator Password:**

After reboot, you will be prompted to set the built-in Administrator password.

| Requirement       | Details                                          |
| ----------------- | ------------------------------------------------ |
| Minimum length    | 8 characters (14+ recommended)                   |
| Must contain      | Uppercase, lowercase, number, special character  |
| Suggestion        | `LabAdmin2026!` *(easy to remember for testing)* |

Enter the password twice, then click **Finish**.

> 📝 **Write down this password — you will need it to log in.**

---

## 4.4 First Login

1. Press **Ctrl+Alt+Delete** *(In VirtualBox: Input → Keyboard → Insert Ctrl+Alt+Del)*
2. Username: `Administrator` *(pre-filled)*
3. Enter your password and press **Enter**
4. Wait for the desktop to load *(first login takes 2–3 minutes)*
5. **Server Manager** will open automatically

> ✅ **Windows Server 2022 is now installed.** Next, we will configure the server and install Active Directory.

---

# 5. Initial Server Configuration

## 5.1 Configure Static IP Address

Before installing Active Directory, you must assign a static IP address.

**Step 1 — Open Network Settings:**

1. Click **Start → Settings** (gear icon)
2. Click **Network & Internet**
3. Click **Ethernet** (left sidebar)
4. Click on **Ethernet** or **Ethernet 2** *(the Internal Network adapter)*
5. Scroll down and click **Edit** under *IP assignment*

**Step 2 — Configure IPv4 Settings:**

Change from *Automatic (DHCP)* to **Manual**, then configure:

| Setting                | Value                                              |
| ---------------------- | -------------------------------------------------- |
| IPv4 Toggle            | ON                                                 |
| IP Address             | `192.168.20.10`                                    |
| Subnet prefix length   | `24`                                               |
| Gateway                | `192.168.20.1` *(pfSense Server_Network interface)*|
| Preferred DNS          | `127.0.0.1` *(itself, after AD DNS is installed)*  |
| Alternate DNS          | `8.8.8.8` *(Google DNS as backup)*                 |

Click **Save**

> ⚠️ **My Lab Note:** If your setup uses `192.168.3.x/24`, set the IP to your DC's actual address (e.g., `192.168.3.10`) and gateway to `192.168.3.1`.

**Step 3 — Verify Network Connectivity:**

Open PowerShell as Administrator *(Start → type "PowerShell" → right-click → Run as Administrator)*:

```powershell
Test-NetConnection google.com
```

Expected result: `PingSucceeded: True`

---

## 5.2 Rename the Server

Give the server a recognizable name before promoting it to Domain Controller.

1. Open **Server Manager** → Click **Local Server** (left sidebar)
2. Click on the computer name *(currently shows a random name like `WIN-XXXX`)*
3. In the **System Properties** window, click **Change...**
4. Set **Computer name:** `DC01`
5. Click **OK** twice
6. **Restart** when prompted

After restart, log back in with the `Administrator` account.

---

## 5.3 Install Windows Updates *(Optional but Recommended)*

1. Open **Settings → Windows Update**
2. Click **Check for updates**
3. Install all available updates and restart if required
4. Repeat until no more updates are available

> ⏳ This can take **30–60 minutes**. You can skip this and continue with AD installation, but updates are recommended before domain promotion.

---

# 6. Active Directory Domain Services Installation

## 6.1 Install AD DS Role

**Step 1 — Add Roles and Features:**

1. Open **Server Manager**
2. Click **Manage** (top right) → **Add Roles and Features**
3. The *Add Roles and Features Wizard* opens
4. Click **Next** *(Before You Begin page)*
5. Installation Type: Select **"Role-based or feature-based installation"** → Click **Next**
6. Server Selection: Select **DC01** *(should be pre-selected)* → Click **Next**

**Step 2 — Select Active Directory Domain Services:**

1. On the *Server Roles* page, scroll down and check **"Active Directory Domain Services"**
2. A popup appears: *"Add features that are required for Active Directory Domain Services?"*
3. Click **"Add Features"**
4. Verify **"Active Directory Domain Services"** is checked → Click **Next**

**Step 3 — Features:**
Keep all defaults → Click **Next**

**Step 4 — AD DS Information:**
Read the information page → Click **Next**

**Step 5 — Confirmation:**

1. Review your selections
2. Optionally check **"Restart the destination server automatically if required"**
3. Click **Install** *(takes 3–5 minutes)*
4. Wait for: *"Installation succeeded on DC01"*
5. Click **Close** — do **NOT** restart yet

> ⚠️ **Important:** The AD DS role is now installed, but the server is **not yet a Domain Controller**. The next step promotes it.

---

## 6.2 Promote Server to Domain Controller

**Step 1 — Start Domain Controller Promotion:**

1. In Server Manager, click the **yellow warning flag (⚠)** in the top menu bar
2. Click **"Promote this server to a domain controller"**
3. The *Active Directory Domain Services Configuration Wizard* opens

**Step 2 — Deployment Configuration:**

| Setting                    | Value                  |
| -------------------------- | ---------------------- |
| Deployment operation       | **Add a new forest**   |
| Root domain name           | `lab.local`            |

> **Why `lab.local`?** This is common practice for lab environments — it's not a real internet domain and won't conflict with anything. Alternatives: `corp.local`, `test.local`, `homelab.local`

Click **Next**

**Step 3 — Domain Controller Options:**

| Setting                          | Value                                            |
| -------------------------------- | ------------------------------------------------ |
| Forest functional level          | Windows Server 2016 *(default)*                  |
| Domain functional level          | Windows Server 2016 *(default)*                  |
| DNS server                       | ✅ Checked *(CRITICAL — leave this checked)*     |
| Global Catalog (GC)              | ✅ Checked *(auto-selected, leave it)*           |
| Read-only domain controller      | ☐ Unchecked *(this is a writable DC)*            |
| DSRM password                    | Enter and confirm a strong password              |

> 📝 **Write down the DSRM password** — it is needed for disaster recovery.

Click **Next**

**Step 4 — DNS Options:**

You may see: *"A delegation for this DNS server cannot be created..."*
This is **normal** and can be safely ignored in a lab environment. Click **Next**

**Step 5 — Additional Options:**

| Setting             | Value                                               |
| ------------------- | --------------------------------------------------- |
| NetBIOS domain name | `LAB` *(auto-populated from `lab.local`)*           |

> This is the short name users will see (e.g., `LAB\Administrator`). Click **Next**

**Step 6 — Paths:**

Keep all default paths:

| Path             | Default Location         |
| ---------------- | ------------------------ |
| Database folder  | `C:\Windows\NTDS`        |
| Log files folder | `C:\Windows\NTDS`        |
| SYSVOL folder    | `C:\Windows\SYSVOL`      |

Click **Next**

**Step 7 — Review Options:**

Review all selections:
- Forest and domain: `lab.local`
- DNS server: Yes
- NetBIOS name: `LAB`

> 💡 Optionally click **"View script"** to see the equivalent PowerShell commands — useful for learning.

Click **Next**

**Step 8 — Prerequisites Check:**

The wizard runs a prerequisites check *(1–2 minutes)*. You may see yellow warnings — these are safe to ignore:

| Warning                        | Action           |
| ------------------------------ | ---------------- |
| DNS delegation warning         | ✅ Ignore        |
| Default first site warning     | ✅ Ignore        |
| Windows Server 2016 warning    | ✅ Ignore        |

> ❌ If you see **red errors**, do **NOT** continue — fix the errors first.

If only yellow warnings: Click **Install**

**Step 9 — Installation and Automatic Restart:**

> ⏳ Installation takes **5–10 minutes**. The progress bar shows *"Configuring Active Directory Domain Services"*. The server will **restart automatically** — do not close the window or power off the VM.

After restart, the login screen will show `LAB\Administrator` — notice the domain prefix!

---

## 6.3 First Login to Domain

1. After the automatic restart, the VM boots into the domain
2. Login screen shows: `LAB\Administrator`
3. Press **Ctrl+Alt+Delete** and enter your Administrator password *(same as before)*
4. Wait for login *(first domain login takes 2–3 minutes)*
5. **Active Directory Users and Computers** is now available under **Tools**

> ✅ **You now have a functioning Active Directory domain controller!**

---

## 6.4 Verify Active Directory Installation

**Verify AD is working:**

1. Open **Server Manager → Tools → Active Directory Users and Computers**
2. Expand `lab.local` in the left tree
3. You should see default containers:

   `Builtin` · `Computers` · `Domain Controllers` · `ForeignSecurityPrincipals` · `Managed Service Accounts` · `Users`

4. Click **Domain Controllers** → `DC01` should be listed

**Verify DNS:**

Open Command Prompt or PowerShell and run:

```powershell
nslookup lab.local
```

Expected result: `192.168.20.10` *(the DC's IP address)*

# 7. Basic Domain Configuration

## 7.1 Understanding Active Directory Structure

Before creating users, understand the AD hierarchy:

| Level                      | Description                                          |
| -------------------------- | ---------------------------------------------------- |
| **Forest**                 | Top-level container (`lab.local`)                    |
| **Domain**                 | Logical partition within the forest (`lab.local`)    |
| **Organizational Unit (OU)**| Container for organizing users, computers, groups   |
| **Users**                  | User accounts within the domain                      |
| **Computers**              | Domain-joined machines                               |

---

## 7.2 Create Organizational Units

Organizational Units (OUs) help organize objects in AD. We'll create a basic structure.

**Step 1 — Open Active Directory Users and Computers (ADUC):**

- **Server Manager → Tools → Active Directory Users and Computers**

**Step 2 — Create the first OU:**

1. Right-click on `lab.local` → **New → Organizational Unit**
2. Name: `LAB Users`
3. Uncheck **"Protect container from accidental deletion"** *(for lab flexibility)*
4. Click **OK**

**Step 3 — Repeat for the remaining OUs** *(all under `lab.local`)*:

| OU Name         |
| --------------- |
| `LAB Computers` |
| `LAB Servers`   |
| `LAB Admins`    |
| `LAB Groups`    |

---

Your AD structure should now look like this:

```
lab.local
├─ Builtin
├─ Computers
├─ Domain Controllers
├─ LAB Admins
├─ LAB Computers
├─ LAB Groups
├─ LAB Servers
├─ LAB Users
└─ Users
```