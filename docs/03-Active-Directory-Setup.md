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

# 8. Creating Organizational Units & Users

## 8.1 Create Domain Users

Now that Active Directory is configured, we'll create user accounts to simulate an enterprise environment. These users will be used for testing authentication, group policy, and attack scenarios.

**Step 1 — Open Active Directory Users and Computers:**

1. **Server Manager → Tools → Active Directory Users and Computers**
2. Expand `lab.local` in the left tree
3. Navigate to the `LAB Users` OU created earlier

---

**Step 2 — Create Standard User Account:**

1. Right-click `LAB Users` → **New → User**
2. Fill in the form:

   | Field                              | Value     |
   | ---------------------------------- | --------- |
   | First name                         | `John`    |
   | Last name                          | `Doe`     |
   | User logon name                    | `jdoe`    |
   | User logon name (pre-Windows 2000) | `LAB\jdoe`|

3. Click **Next**
4. Configure password:

   | Setting                                  | Value              |
   | ---------------------------------------- | ------------------ |
   | Password                                 | `ComplexPass123!`  |
   | Confirm password                         | `ComplexPass123!`  |
   | User must change password at next logon  | ☐ Unchecked        |
   | Password never expires                   | ✅ Checked         |

5. Click **Next** → Review summary → Click **Finish**

> ⚠️ **Note:** In production, you would **never** disable password expiration or skip password change requirements. We do this in the lab for convenience during testing.

---

**Step 3 — Create Additional Users** *(all in `LAB Users` OU, password: `ComplexPass123!`)*:

| Full Name      | Username     | Purpose                                  |
| -------------- | ------------ | ---------------------------------------- |
| Jane Smith     | `jsmith`     | Standard user — IT Department            |
| Bob Johnson    | `bjohnson`   | Standard user — HR Department            |
| Alice Williams | `awilliams`  | Standard user — Finance                  |
| SQL Service    | `svc_sql`    | Service account *(for Kerberoasting tests)* |

---

**Step 4 — Create Administrator Account** *(in `LAB Admins` OU)*:

1. Right-click `LAB Admins` → **New → User**
2. Configure:

   | Field             | Value           |
   | ----------------- | --------------- |
   | First name        | `Admin`         |
   | Last name         | `User`          |
   | User logon name   | `labadmin`      |
   | Password          | `AdminPass2026!`|
   | Password never expires | ✅ Checked |
   | Must change at next logon | ☐ Unchecked |

3. Click **Next** → **Finish**

---

## 8.2 Create Security Groups

Security groups control access to resources. We'll create groups for different departments.

**Step 1 — Create IT Admins Group:**

1. Navigate to `LAB Groups` OU
2. Right-click → **New → Group**
3. Configure:

   | Field        | Value       |
   | ------------ | ----------- |
   | Group name   | `IT Admins` |
   | Group scope  | Global      |
   | Group type   | Security    |

4. Click **OK**

---

**Step 2 — Create Additional Groups** *(all in `LAB Groups`, Global Security type)*:

| Group Name           | Purpose                     |
| -------------------- | --------------------------- |
| `HR Department`      | Human resources users       |
| `Finance Department` | Finance users               |
| `IT Department`      | IT users                    |
| `Remote Desktop Users` | RDP access testing        |

---

**Step 3 — Add Users to Groups:**

1. Double-click the `IT Admins` group → go to **Members** tab
2. Click **Add...** → type `labadmin` → click **Check Names** → click **OK** twice

Repeat for the following memberships:

| User        | Group(s)                          |
| ----------- | --------------------------------- |
| `jsmith`    | `IT Department`                   |
| `bjohnson`  | `HR Department`                   |
| `awilliams` | `Finance Department`              |
| `labadmin`  | `IT Admins`, `IT Department`      |

---

**Step 4 — Make `labadmin` a Domain Admin:**

1. In ADUC, navigate to the **Builtin** container
2. Double-click **Domain Admins** group → **Members** tab → **Add**
3. Type `labadmin` → click **Check Names** → **OK** → **OK**

> ⚠️ **Warning:** In a real environment, limit Domain Admin membership strictly. We do this in the lab for testing purposes only.

---

# 9. Group Policy Basics

## 9.1 Understanding Group Policy

Group Policy Objects (GPOs) configure and enforce settings across the domain. They are a primary target for attackers (GPO abuse) and essential for SOC monitoring.

**Common GPO use cases:**

| Category              | Examples                                        |
| --------------------- | ----------------------------------------------- |
| Account Security      | Password policies, account lockout              |
| Audit & Logging       | What events to log (critical for Wazuh)         |
| Software Management   | Installation, updates, restrictions             |
| Desktop Configuration | Restrictions, wallpaper, shortcuts              |
| Security Settings     | Firewall rules, UAC, registry values            |

---

## 9.2 Access Group Policy Management

1. Open **Server Manager → Tools → Group Policy Management**
2. Expand **Forest: lab.local → Domains → lab.local**
3. You should see:
   - `Default Domain Policy` *(pre-configured)*
   - `Domain Controllers` OU with `Default Domain Controllers Policy`

---

## 9.3 Configure Default Domain Password Policy

**Step 1 — Edit Default Domain Policy:**

1. In Group Policy Management, expand `lab.local`
2. Right-click **Default Domain Policy** → **Edit**
3. The *Group Policy Management Editor* opens

**Step 2 — Configure Password Policy:**

Navigate to:
`Computer Configuration → Policies → Windows Settings → Security Settings → Account Policies → Password Policy`

| Setting                                    | Value      |
| ------------------------------------------ | ---------- |
| Enforce password history                   | 5 passwords|
| Maximum password age                       | 90 days    |
| Minimum password age                       | 1 day      |
| Minimum password length                    | 12 characters |
| Password must meet complexity requirements | Enabled    |
| Store passwords using reversible encryption| Disabled   |

Double-click each setting to modify it.

**Step 3 — Configure Account Lockout Policy:**

Navigate to:
`Account Policies → Account Lockout Policy`

| Setting                              | Value       |
| ------------------------------------ | ----------- |
| Account lockout duration             | 30 minutes  |
| Account lockout threshold            | 5 invalid attempts |
| Reset account lockout counter after  | 30 minutes  |

Close the **Group Policy Management Editor**.

---

## 9.4 Enable Advanced Audit Policies

Advanced audit policies generate the event logs that Wazuh will monitor. This is **critical for detecting attacks**.

**Step 1 — Edit Default Domain Policy:**

Right-click **Default Domain Policy** → **Edit**

**Step 2 — Configure Audit Policies:**

Navigate to:
`Computer Configuration → Policies → Windows Settings → Security Settings → Advanced Audit Policy Configuration → Audit Policies`

Set each of the following to **Success and Failure**:

| Category             | Policy                              |
| -------------------- | ----------------------------------- |
| Account Logon        | Audit Credential Validation         |
| Account Management   | Audit User Account Management       |
| Account Management   | Audit Security Group Management     |
| Logon/Logoff         | Audit Logon                         |
| Logon/Logoff         | Audit Logoff                        |
| Logon/Logoff         | Audit Account Lockout               |
| Object Access        | Audit File Share                    |
| Policy Change        | Audit Policy Change                 |
| Privilege Use        | Audit Sensitive Privilege Use       |
| System               | Audit Security System Extension     |

Close the **Group Policy Management Editor**.

**Step 3 — Force Group Policy Update:**

Open PowerShell as Administrator and run:

```powershell
gpupdate /force
```

> This applies the policies immediately instead of waiting for the next automatic update cycle.

# 10. Install Wazuh Agent

## 10.1 Why Wazuh Agent on Domain Controller?

The Wazuh agent monitors the Domain Controller for security events and sends them to the Wazuh Manager for analysis. This is essential for detecting Active Directory attacks.

**Events the Wazuh agent will detect:**

| Category                  | Examples                                              |
| ------------------------- | ----------------------------------------------------- |
| Authentication            | Failed login attempts, Kerberos ticket requests       |
| Account Management        | Account creation/deletion, group membership changes   |
| Privilege & Policy        | Privilege escalation, Group Policy modifications      |
| Lateral Movement          | NTLM authentication (Pass-the-Hash)                   |
| Suspicious Activity       | Malicious PowerShell commands                         |

---

## 10.2 Download Wazuh Agent

On the Domain Controller (`DC01`), open a web browser and download directly:

```
https://packages.wazuh.com/4.x/windows/wazuh-agent-4.7.5-1.msi
```

Or visit the official [Wazuh Windows Agent Documentation](https://documentation.wazuh.com) for the latest version.

Save the `.msi` file to your Downloads folder.

---

## 10.3 Install Wazuh Agent

**Step 1 — Run the Installer:**

1. Locate `wazuh-agent-4.7.5-1.msi` in your Downloads folder
2. Double-click to run the installer
3. Click **Next** on the welcome screen

**Step 2 — Configure Agent:**

| Setting             | Value            |
| ------------------- | ---------------- |
| Wazuh server address| `192.168.3.81`   |
| Wazuh server port   | `1514` *(default)*|
| Protocol            | TCP *(default)*  |
| Agent name          | `DC01`           |

Click **Next**

**Step 3 — Complete Installation:**

1. Click **Install**
2. Wait for installation to complete *(1–2 minutes)*
3. Click **Finish**

---

## 10.4 Register Agent with Wazuh Manager

The agent must be authenticated with the Wazuh Manager before it can send logs.

**Step 1 — Open PowerShell as Administrator:**

Start → type `PowerShell` → right-click → **Run as Administrator**

**Step 2 — Navigate to Wazuh directory:**

```powershell
cd "C:\Program Files (x86)\ossec-agent"
```

**Step 3 — Register the agent:**

```powershell
.\agent-auth.exe -m 192.168.3.81
```

Expected output:

```
INFO: Connected to enrollment service.
INFO: Using agent name as: DC01
INFO: Valid key received.
```

**Step 4 — Start Wazuh Agent service:**

```powershell
NET START WazuhSvc
```

Expected output: `"The Wazuh service was started successfully."`

---

## 10.5 Verify Agent Connection

On the Wazuh Manager (`192.168.3.81`), SSH in and run:

```bash
sudo /var/ossec/bin/agent_control -lc
```

You should see `DC01` listed as **Active**:

```
ID: 002, Name: DC01, IP: 192.168.20.10, Active/Online
```

> ✅ **DC01 is now sending logs to Wazuh Manager.**

---

# 11. Install & Configure Sysmon

## 11.1 What is Sysmon?

System Monitor (Sysmon) is a Windows system service that logs detailed information about system activity — far beyond what Windows Event Logs capture by default.

**What Sysmon monitors:**

| Category              | Details                                         |
| --------------------- | ----------------------------------------------- |
| Process Activity      | Creation, termination, command-line arguments   |
| Network Connections   | Source/destination IPs and ports                |
| File System           | File creation and modification                  |
| Registry              | Registry key changes                            |
| Driver & Module Load  | Loaded drivers and DLLs                         |
| DNS Queries           | All DNS lookups made by processes               |
| Process Access        | Process injection attempts                      |
| Clipboard             | Clipboard capture events                        |

**Why Sysmon is critical for a SOC lab:**
- Detects malware execution patterns
- Tracks lateral movement techniques
- Identifies credential theft attempts
- Logs full command-line arguments *(catches malicious scripts)*
- Provides detailed forensic evidence
- Integrates with Wazuh for real-time alerting

---

## 11.2 Download Sysmon

Sysmon is part of Microsoft Sysinternals.

**Official download:**

```
https://download.sysinternals.com/files/Sysmon.zip
```

Or visit the [Microsoft Sysinternals Sysmon Documentation](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon) for the latest version.

---

## 11.3 Download SwiftOnSecurity Sysmon Configuration

SwiftOnSecurity maintains the industry-standard Sysmon configuration file used by most SOC teams.

**Download link:**

```
https://github.com/SwiftOnSecurity/sysmon-config
```

1. Click the **Raw** button on the `sysmonconfig-export.xml` file to download it
2. Save the file as: `sysmonconfig.xml`
3. Place it in `C:\Tools\` *(create the folder if needed)*

---

## 11.4 Install Sysmon

**Step 1 — Extract Sysmon:**

1. Extract `Sysmon.zip` to `C:\Tools\Sysmon\`
2. Verify these files are present: `Sysmon64.exe`, `Sysmon.exe`, `Eula.txt`

**Step 2 — Install with SwiftOnSecurity config:**

Open PowerShell as Administrator and run:

```powershell
cd C:\Tools\Sysmon
.\Sysmon64.exe -accepteula -i C:\Tools\sysmonconfig.xml
```

| Flag            | Purpose                                       |
| --------------- | --------------------------------------------- |
| `-accepteula`   | Accepts license agreement automatically       |
| `-i`            | Install with the specified configuration file |
| `sysmonconfig.xml` | SwiftOnSecurity's detection rules          |

Expected output:

```
Sysmon64 installed.
SysmonDrv installed.
```

**Step 3 — Verify installation:**

```powershell
Get-Service Sysmon64
```

Expected: `Status = Running`

---

## 11.5 Configure Wazuh to Monitor Sysmon Logs

Tell the Wazuh agent to collect Sysmon events.

**Step 1 — Edit Wazuh agent configuration:**

```powershell
notepad "C:\Program Files (x86)\ossec-agent\ossec.conf"
```

**Step 2 — Add Sysmon log collection:**

Find the `<localfile>` section and add this block:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

Save and close Notepad.

**Step 3 — Restart Wazuh agent:**

```powershell
Restart-Service WazuhSvc
```

---

## 11.6 Test Sysmon Logging

**Generate a test event:**

1. Open PowerShell and run `notepad.exe`, then close it
2. Open **Event Viewer:** `eventvwr.msc`
3. Navigate to:
   `Applications and Services Logs → Microsoft → Windows → Sysmon → Operational`
4. You should see **Event ID 1** *(Process Create)* for `notepad.exe`

**Verify events are reaching Wazuh Manager:**

```bash
sudo tail -f /var/ossec/logs/alerts/alerts.json | grep -i sysmon
```

> ✅ If you see Sysmon events in the output, the full pipeline — **DC01 → Wazuh Agent → Wazuh Manager** — is working correctly.

# 12. Verification & Testing

## 12.1 Verify Active Directory Services

Run these PowerShell commands to verify all AD services are running:

```powershell
Get-Service ADWS    | Select Name, Status
Get-Service DNS     | Select Name, Status
Get-Service NTDS    | Select Name, Status
Get-Service KDC     | Select Name, Status
Get-Service Netlogon | Select Name, Status
```

All should show `Status: Running`.

| Service    | Role                                  |
| ---------- | ------------------------------------- |
| `ADWS`     | Active Directory Web Services         |
| `DNS`      | Domain Name System                    |
| `NTDS`     | AD Domain Services (core database)    |
| `KDC`      | Kerberos Key Distribution Center      |
| `Netlogon` | Domain authentication and replication |

---

## 12.2 Test Domain Authentication

1. Press **Windows + L** to lock the screen
2. Click **Other user**
3. Enter credentials:

   | Field    | Value             |
   | -------- | ----------------- |
   | Username | `LAB\jdoe`        |
   | Password | `ComplexPass123!` |

4. Login should succeed

---

## 12.3 Verify DNS Resolution

```powershell
nslookup lab.local
nslookup DC01.lab.local
nslookup _ldap._tcp.lab.local    # Service record test
```

All three should resolve to `192.168.20.10`.

---

## 12.4 Verify Group Policy Application

```powershell
gpresult /R
```

This shows which GPOs are applied to the computer and current user.

---

## 12.5 Check Wazuh Alerts

On the Wazuh Manager (`192.168.3.81`):

**Verify DC01 is active:**

```bash
sudo /var/ossec/bin/agent_control -lc
```

`DC01` should show as **Active**.

**Check recent events from DC01:**

```bash
sudo tail -100 /var/ossec/logs/alerts/alerts.json | grep DC01
```

> ✅ If you see recent events, the Domain Controller is successfully sending logs to Wazuh.

---

# 13. Security Hardening

## 13.1 Disable Unnecessary Services

While this is a lab, practicing security hardening is valuable:

| Service          | Safe to Disable? | Reason                                        |
| ---------------- | ---------------- | --------------------------------------------- |
| Print Spooler    | ✅ Yes *(lab only)* | Common attack vector *(PrintNightmare)*    |
| Windows Update   | ❌ No            | Keep enabled for security patches             |
| Remote Registry  | ✅ Yes           | Rarely needed, reduces attack surface         |
| Server service   | ❌ No            | Required for file sharing and AD              |

To disable a service:

```powershell
Stop-Service "Spooler"
Set-Service "Spooler" -StartupType Disabled
```

---

## 13.2 Enable Windows Firewall Logging

Log dropped connections for later analysis:

```powershell
Set-NetFirewallProfile -Profile Domain,Public,Private -LogBlocked True
```

Logs are saved to:

```
C:\Windows\System32\LogFiles\Firewall\pfirewall.log
```

---

## 13.3 Disable SMBv1

SMBv1 is vulnerable to critical exploits *(WannaCry, NotPetya)*. Disable it:

```powershell
Disable-WindowsOptionalFeature -Online -FeatureName SMB1Protocol
```

---

## 13.4 Enable Protected Users Group

Add high-value accounts (like `labadmin`) to the **Protected Users** group. This prevents NTLM authentication and credential caching — making Pass-the-Hash attacks significantly harder.

1. Open **Active Directory Users and Computers**
2. Navigate to the **Builtin** container
3. Double-click **Protected Users** group → **Members** tab → **Add**
4. Type `labadmin` → **Check Names** → **OK** → **OK**

---

## 13.5 Disable NAT Adapter *(Post-Setup)*

After all installations and updates are complete, disable Adapter 2 (NAT) so the Domain Controller only has internal network connectivity.

1. Shut down the `DC01` VM
2. Go to **VirtualBox → DC01 → Settings → Network → Adapter 2**
3. Uncheck **"Enable Network Adapter"**
4. Click **OK**
5. Start the `DC01` VM

---

# 14. Common Troubleshooting

## 14.1 Domain Controller Won't Promote

**Problem:** Prerequisites check fails during DC promotion

| Cause                         | Fix                                              |
| ----------------------------- | ------------------------------------------------ |
| Static IP not configured      | Set static IP before starting promotion          |
| DNS pointing to external server | Change DNS to `127.0.0.1`                      |
| Computer name is still default | Rename server to `DC01` before promotion        |
| Firewall blocking ports       | Temporarily disable Windows Firewall during setup|

---

## 14.2 DNS Not Working

**Problem:** Cannot resolve `lab.local` or `DC01.lab.local`

```powershell
Get-Service DNS                  # Verify DNS is running
Restart-Service DNS              # Restart if needed
ipconfig /flushdns               # Clear DNS cache
```

Additional checks:
- Confirm the DNS server address in adapter settings is `127.0.0.1`
- Verify DNS zones exist: **Server Manager → Tools → DNS → Forward Lookup Zones**

---

## 14.3 Users Cannot Log In

**Problem:** Domain users get *"Username or password incorrect"* error

```powershell
Get-ADUser -Identity jdoe                        # Verify user exists
Get-ADUser jdoe | Select Enabled                 # Check account is enabled
Get-ADUser jdoe -Properties LockedOut            # Check for lockout
```

Additional checks:
- Use the full domain prefix: `LAB\username` *(not just `username`)*
- Verify the NetBIOS name: **Server Manager → Local Server → Computer Name**
- Test with the built-in Administrator first: `LAB\Administrator`

---

## 14.4 Wazuh Agent Not Connecting

**Problem:** DC01 shows as *"Never connected"* or *"Disconnected"* in Wazuh

```powershell
Get-Service WazuhSvc                                          # Check service status
Test-NetConnection 192.168.3.81 -Port 1514                    # Test connectivity
notepad "C:\Program Files (x86)\ossec-agent\ossec.conf"       # Verify IP in config
```

If the above looks correct, re-register and restart:

```powershell
cd "C:\Program Files (x86)\ossec-agent"
.\agent-auth.exe -m 192.168.3.81
Restart-Service WazuhSvc
```

> Also check **pfSense firewall rules** — ensure `Server_Network → DMZ` traffic is allowed on port `1514`.

---

## 14.5 Sysmon Events Not Appearing in Wazuh

**Problem:** Sysmon is installed but no events appear in Wazuh

```powershell
Get-Service Sysmon64    # Verify Sysmon is running
eventvwr.msc            # Check Event Viewer: Microsoft-Windows-Sysmon/Operational
```

Checklist:

- [ ] Sysmon service status shows `Running`
- [ ] Event Viewer shows Sysmon events under `Microsoft-Windows-Sysmon/Operational`
- [ ] `ossec.conf` contains the Sysmon `<localfile>` block *(see Section 11.5)*
- [ ] Wazuh agent was restarted after the config change
- [ ] Check Wazuh agent log for errors: `C:\Program Files (x86)\ossec-agent\ossec.log`