*This project has been created as part of the 42 curriculum by jbarreir.*

# Born2beRoot

This project consists of creating a strictly configured Debian server on a Virtual Machine. It focuses on system administration, security (Sudo, UFW, Password policies), and automated monitoring scripts.

## Description

### Operating System: Debian vs. Rocky Linux

In the initial phase of the project, a choice had to be made between two major Linux distributions: Debian and Rocky Linux.

The fundamental difference lies in their philosophy and lineage: **Debian** is a community-led, independent project renowned for its "Universal OS" status and legendary stability. In contrast, **Rocky Linux** was created by one of the founders of CentOS as a direct successor to it, after Red Hat shifted CentOS towards a "Stream" (rolling-release) model. Its primary goal is to provide a 1:1 binary-compatible, enterprise-grade alternative to RHEL (Red Hat Enterprise Linux).

| Feature | Debian | Rocky Linux |
| :---     | :---    | :---  |
| **Family** | Community-based (Independent) | RHEL-based (Red Hat) |
| **Package Manager** | apt (.deb) | dnf / yum (.rpm)
| **Release Cycle** | "Ultra-stable, slower updates" | "Enterprise-focused, binary-compatible with RHEL" |
| **Focus** | Stability and Free Software philosophy | Stability and corporate/server environments |

#### Why Debian?

I chose Debian for this project based on the following specific criteria:

- **Subject Recommendation**: The project subject explicitly suggests Debian for those discovering virtualization for the first time. Its straightforward installation and vast documentation provide a more accessible learning curve for initial system administration.

- **ARM64 Stability** (Apple Silicon): While Rocky Linux carries the robust CentOS heritage for corporate environments, Debian’s long-standing reputation for stability on diverse hardware ensures a more reliable and simpler experience when virtualizing on Apple Silicon.

- **Community Support**: As an independent, community-driven project, Debian has a massive user base. This ensures that any issue encountered during the configuration of services like SSH or UFW can be quickly resolved through well-documented solutions.

## Instructions

### VirtualBox

#### Hypervisors: VirtualBox vs. UTM
Since this project was developed on **Apple Silicon (ARM64)**, the choice of hypervisor was a strategic decision between performance and learning value:

* **UTM:** * **Nature:** A lightweight wrapper for Apple's native Virtualization.Framework and QEMU.
    * **Pros:** Offers near-native performance on M1/M2/M3 chips through hardware acceleration.
* **VirtualBox (Version 7.x):**
    * **Nature:** A classic **Type 2 Hypervisor**.
    * **Pros:** Highly standardized, robust interface, and advanced virtual networking capabilities.
    * **Cons:** On ARM64, it is currently in "Beta" and uses a slower virtualization core compared to native frameworks.

**Why I chose VirtualBox:**
I decided to use **VirtualBox** despite it being less performant on Apple Silicon because it is a **universal industry standard**. Learning to manage a "traditional" hypervisor like VirtualBox provides skills that are directly transferable to Windows and Intel-based Linux environments.

#### VirtualBox Configuration

The virtual machine was configured with the following parameters to ensure compatibility with Apple Silicon (ARM64):
* **Memory:** 1024 MB - Since the server operates without a Graphical User Interface (GUI), 1 GB of RAM is more than sufficient. This allocation ensures smooth performance for background services (SSH, Cron, UFW).
* **Disk Space:** 20.00 GB (VDI) - While the system could technically run on a 10 GB disk, 20 GB was chosen to ensure long-term stability. This extra space accounts for the overhead required by **LVM encryption (LUKS)** and provides a safety margin for the mandatory **Sudo log files** and kernel updates, preventing "disk full" errors that could compromise the server's availability.
* **Network:** Bridged Adapter or NAT with Port Forwarding (Port 4242 for SSH).

### Debian Installation Steps

#### LVM and Partitioning Scheme
The system follows a professional partitioning standard, combining the **UEFI** boot process with **LVM** (Logical Volume Management) over an **encrypted (LUKS)** layer. 

| Partition | Type | Mount Point | Size | Description |
| :--- | :--- | :--- | ---: | :--- |
| `sda1` | Primary | `/boot/efi` | 100MB | EFI System Partition |
| `sda2` | Primary | `/boot` | 500MB | Bootloader files |
| `sda3` | Primary | `crypt` | 20GB | Encrypted Physical Volume (LUKS) |
| `LVMGroup-root` | Logical | `/` | 9GB | Root filesystem |
| `LVMGroup-swap` | Logical | `[SWAP]` | 1GB | Swap space |

#### Why use EFI (`/boot/efi`)?
Modern systems use **UEFI** (Unified Extensible Firmware Interface) instead of the legacy BIOS.
* **Standardization:** It is the current industry standard and is required for modern hardware and virtualization on **ARM64** (Apple Silicon) architectures.
* **Function:** This partition (formatted as FAT32) contains the bootloader executables. It is the first point of contact for the firmware to initiate the OS boot process.

#### The Boot Partition (`/boot`)
* **Function:** It stores the Linux Kernel, the `initrd` image, and GRUB configuration files.
* **Necessity:** Since the rest of the system is encrypted, the bootloader needs an unencrypted space to find the instructions and kernel files required to start the decryption process.

#### Encrypted Volume (`sda3_crypt`)
* **Security:** Using **LUKS** (Linux Unified Key Setup), the entire Physical Volume is encrypted. 
* **Data Protection:** This ensures that if the virtual disk (.vdi) is accessed externally or stolen, the data remains unreadable without the decryption passphrase.

#### Logical Volume Management (LVM)
Within the encrypted volume, I established a **Volume Group** (`LVMGroup`) to manage space dynamically:
* **Root (`/`):** The main directory where the OS, system utilities, and mandatory sudo logs are stored.
* **Swap:** Used as virtual memory. Placing it *inside* the encrypted LVM is a security best practice, ensuring that any sensitive data temporarily moved from RAM to disk remains encrypted. Same size as RAM per best practices.

### User and Root Configuration

During the installation, the Debian installer prompts for the setup of the privileged account (root) and the primary user.

#### The Root Account
* **Role:** The `root` user is the superuser of the system, with absolute permissions.
* **Setup:** A strong password was assigned to the root account. 
* **Note:** In Debian, by setting a root password, the `sudo` package is not installed by default, as the system expects you to use `su -` for administrative tasks. This is a key difference from other distributions that we will address in the Post-Installation section.

#### The Primary User
* **Username:** `jbarreir` (as per the 42 login requirement).
* **Role:** This is a non-privileged user intended for daily tasks.
* **Group:** Initially, this user does not have administrative rights. During the post-installation, we will manually install `sudo` and add `jbarreir` to the `sudo` and `user42` groups to comply with the project's security policy.

#### Security Implications
By separating the `root` account from the primary user from the start:
* We follow the **Principle of Least Privilege**. This security practice ensures that users and programs have only the necessary access required to perform their tasks.
* We ensure that administrative actions require a conscious escalation of privileges.
* We prepare the system for a custom **Sudo Policy** which will log every action performed by the user.

### Software Selection and Services

During the Debian installation process, a minimalist approach was taken to comply with the project's requirements. **No Desktop Environment** was installed. Only "SSH Server" and "Standard System Utilities" were selected.

In the `tasksel` (Software Selection) step, the following choices were made:
* **[ ] Debian desktop environment:** **Deselected**. The server must operate in CLI (Command Line Interface) mode only.
* **[ ] GNOME / XFCE / etc:** **Deselected**. Installing a GUI (Graphical User Interface) violates the project's constraints and wastes resources.
* **[X] SSH server:** **Selected**. Essential for remote administration (later configured to port 4242).
* **[X] Standard system utilities:** **Selected**. Provides basic tools for system management (shell, file editors, etc.).

## Post-Installation Configuration

### Sudo Implementation and Policy

Since Debian was installed with a root password, `sudo` had to be installed and configured manually to comply with the project's security standards.

#### Installation & Setup:
1. **Switch to root:** `su -`
2. **Install sudo:** `apt update && apt install sudo`
3. **Add user to sudo group:** `adduser jbarreir sudo`
4. **Create a custom group:** `groupadd user42` and `adduser jbarreir user42`

#### Custom Sudo Policy:
The project requires a strict security policy. This was configured by editing the sudoers file with `visudo` at `/etc/sudoers.d/sudo_config`:

```bash
Defaults    passwd_tries=3
Defaults    badpass_message="Wrong password! Try again."
Defaults    logfile="/var/log/sudo/sudo_log"
Defaults    log_input, log_output
Defaults    iolog_dir="/var/log/sudo"
Defaults    requiretty
```

To enhance system security and accountability, a strict sudoers policy was implemented. Below are the specific requirements and their technical justifications:

* **Authentication Limit:** Restricted to **3 attempts** in case of password error.
    * *Purpose:* Mitigates brute-force attacks on the user's password.
* **Custom Error Message:** A personalized message is displayed when an incorrect password is entered.
    * *Implementation:* Configured via the `badpass_message` parameter.
* **Archiving (Logfile):** Every action performed via `sudo` is recorded in `/var/log/sudo/sudo_log`.
    * *Purpose:* Maintains a centralized audit trail of all administrative commands.
* **Input/Output Logging:** Every command entered and its subsequent output is stored in the `/var/log/sudo` directory.
    * *Purpose:* Provides a full session replay for auditing, ensuring that both the intent (input) and the result (output) of a command are documented.
* **TTY Mode:** `sudo` can only be run from a real terminal (**TTY**).
    * *Purpose:* Prevents the execution of privileged commands from background scripts or unauthorized shells, ensuring a human-in-the-loop interaction.

## Network Security: UFW & SSH

To secure the server's communications, I implemented a firewall policy and hardened the Secure Shell (SSH) configuration.

### A. UFW (Uncomplicated Firewall)
UFW was installed to manage incoming and outgoing traffic, following the "Deny by Default" principle.

* **Installation:** `apt install ufw`
* **Default Policy:** All incoming connections are blocked by default.
* **Specific Rule:** Only port **4242** is allowed to enable SSH access.
* **Verification:**
```bash
sudo ufw enable
sudo ufw allow 4242
sudo ufw status numbered
```
#### UFW vs. firewalld
* **UFW:**
    * **Philosophy:** Designed to be "uncomplicated." It is a user-friendly frontend for `iptables` (or `nftables`).
    * **Usage:** Ideal for standalone servers and simpler rule sets. It is the standard in the Debian/Ubuntu world.
* **firewalld:**
    * **Philosophy:** Uses "zones" to manage traffic based on the trust level of network connections.
    * **Usage:** Standard in RHEL/Rocky Linux. It is more dynamic and suited for complex network environments.

### B. SSH Hardening

The default SSH configuration was modified at `/etc/ssh/sshd_config` to comply with the project's security requirements and prevent unauthorized access:

* **Port Change:** The default port `22` was changed to **`4242`**.
* **Root Login Prevention:** `PermitRootLogin no` was set to prevent direct administrative access via SSH, forcing the use of a standard user account and `sudo`.
* **Service Management:**

```bash
sudo systemctl restart ssh
sudo systemctl status ssh
```

#### Why Port 4242?
Using a non-standard port (anything other than 22) is a basic security practice known as **Security through Obscurity**. It significantly reduces the volume of automated "bot" attacks and brute-force attempts that constantly scan the internet for open port 22 connections.

#### Connection Procedure
To access the server from the host machine, use the following command:
`ssh jbarreir@localhost -p 4242`

> **Note:** This requires VirtualBox Port Forwarding to be configured: **Host 4242 -> Guest 4242**.

### C. Mandatory Access Control (MAC): AppArmor

To enforce security policies, Linux systems use MAC (Mandatory Access Control) kernels. In this project, **AppArmor** is used.Although AppArmor is typically pre-installed in Debian, its status was verified and manually enabled to ensure the system's protection:

1. **Installation:**
   ```bash
   sudo apt update
   sudo apt install apparmor apparmor-utils
   ```
2. **Enabling the Service:**

```Bash
# Enable AppArmor to start at boot
sudo systemctl enable apparmor
sudo systemctl start apparmor
```
3. **Verifying Status:**
To check if AppArmor is active and see how many profiles are loaded:

```Bash
sudo aa-status
```
> (Note: This command shows which processes are in "enforce mode" vs "complain mode").

#### AppArmor vs. SELinux
Both tools provide a security layer that restricts programs' capabilities, but they differ in complexity and approach.

* **AppArmor (Debian's choice):**
    * **Path-based:** It identifies files by their location in the file system.
    * **Simplicity:** It is easier to configure and learn, using "profiles" to allow or deny specific actions.
    * **Status:** It is installed and enabled by default in Debian 12.
* **SELinux (RHEL/Rocky Linux's choice):**
    * **Label-based:** It uses security contexts (labels) assigned to every file, process, and network port.
    * **Complexity:** Much more granular and powerful, but has a steeper learning curve and is harder to troubleshoot.

## Password Policy

A rigorous password policy was implemented to ensure that all user credentials meet high-security standards. This involves two main areas: aging (expiration) and complexity.

### A. Password Aging (Expiration)
Configured in `/etc/login.defs` to limit the lifespan of passwords:

* **PASS_MAX_DAYS: 30** - Passwords must be changed every 30 days to mitigate the risk of compromised credentials.
* **PASS_MIN_DAYS: 2** - Users must wait at least 2 days before changing their password again, preventing them from immediately cycling back to a previous password.
* **PASS_WARN_AGE: 7** - Users receive a warning message 7 days before their password expires.

### B. Password Complexity (PAM)
To enforce complexity, the `libpam-pwquality` package was installed and configured in `/etc/pam.d/common-password`:

```bash
# Installation
sudo apt install libpam-pwquality

# Configuration line added:
password requisite pam_pwquality.so retry=3 minlen=10 ucredit=-1 dcredit=-1 maxrepeat=3 reject_username difok=7 enforce_for_root
```

To ensure passwords are computationally difficult to crack, the following rules were enforced via the `pam_pwquality` module:

* **`minlen=10`**: Establishes a minimum length of 10 characters.
* **`ucredit=-1`**: Requires at least one uppercase letter.
* **`dcredit=-1`**: Requires at least one numerical digit.
* **`maxrepeat=3`**: Prevents the use of more than 3 consecutive identical characters (e.g., "aaaa" is forbidden).
* **`reject_username`**: Checks if the password contains the username in any form, rejecting it if it does.
* **`difok=7`**: Requires that at least 7 characters in the new password were not present in the old one.
* **`enforce_for_root`**: Ensures that all the above complexity rules are also applied to the root user, not just standard users.

### C. Retroactive Application

Changes made to `/etc/login.defs` only apply to users created *after* the modification. To ensure the current accounts comply with the new aging policy, the `chage` command was used:

```bash
# Apply aging policy to root
sudo chage -M 30 -m 2 -W 7 root

# Apply aging policy to the primary user
sudo chage -M 30 -m 2 -W 7 jbarreir
```

## Monitoring Script

The project requires a shell script that displays system information on all terminals every 10 minutes. This script, named `monitoring.sh`, was developed in **Bash** and is scheduled via **Cron**.

### A. Script Components
The script collects and formats the following data points:
* **Architecture:** OS version and kernel version (`uname -a`).
* **Physical Processors:** Number of physical CPUs (`grep "physical id"`).
* **Virtual Processors:** Number of logical cores (`grep "processor"`).
* **Memory Usage:** Used/Total RAM and percentage (`free`).
* **Disk Usage:** Used/Total storage and percentage (`df`).
* **CPU Load:** Current utilization percentage (`top` or `vmstat`).
* **Last Reboot:** Date and time of the last system start (`who -b`).
* **LVM Status:** Whether LVM is active or not (`lsblk`).
* **TCP Connections:** Number of established network connections (`ss`).
* **User Log:** Number of users currently logged in (`users`).
* **Network Info:** IP address and MAC address (`hostname -I` and `ip link`).
* **Sudo Commands:** Total number of commands executed with `sudo` (`journalctl`).


### B. Automation with Cron
To ensure the script runs every 10 minutes from the system's boot, it was added to the **root crontab**:

1. **Access Crontab:** `sudo crontab -e`
2. **Schedule:**
```cron
*/10 * * * * /usr/local/bin/monitoring.sh
```

### C. The code

```bash
#!/bin/bash

#		Arquitecture
	arc=$(uname -srvm | awk '{print $1 " " $2 " " $NF}')

# 		CPU
	pcpu=$(grep "processor" /proc/cpuinfo | wc -l)

#		vCPU
	vcpu=$(grep "^processor" /proc/cpuinfo | wc -l)

#		RAM
	ram_total=$(free -m | awk '$1 == "Mem:" {print $2}')
	ram_use=$(free -m | awk '$1 == "Mem:" {print $3}')
	ram_pct=$(free | awk '$1 == "Mem:" {printf("%.2f"), $3/$2*100}')

#		DISK
	disk_total=$(df -m | grep "/$" | awk '{print $2}')
	disk_use=$(df -m | grep "/$" | awk '{print $3}')
	disk_pct=$(df -m | grep "/$" | awk '{print $5}')

#		CPU Load
	cpul=$(top -bn2 -d 1.0 | grep "Cpu(s)" | tail -1 | awk '{print 100 - $8"%"}')

#		LAST BOOT
	lb=$(who -b | awk '{print $3 " " $4}')

#		LVM
	lvmt=$(lsblk | grep "lvm" | wc -l)
	lvmu=$(if [ $lvmt -gt 0 ]; then echo yes; else echo no; fi)

#		TCP
	tcpc=$(ss -ta | grep ESTAB | wc -l)

#		USER LOG
	ulog=$(users | wc -w)

#		NETWORK
	ip=$(hostname -I | awk '{print $1}')
	mac=$(ip link | grep "link/ether" | awk '{print $2}')

#		SUDO
	cmds=$(journalctl _COMM=sudo | grep COMMAND | wc -l)

#		OUTPUT
	wall "	*** SERVER STATUS ***
	
	#Architecture   : $arc
	#CPU physical   : $pcpu
	#vCPU           : $vcpu
	#Memory Usage   : $ram_use/${ram_total}MB ($ram_pct%)
	#Disk Usage     : $disk_use/${disk_total}MB ($disk_pct)
	#CPU load       : $cpul
	#Last boot      : $lb
	#LVM use        : $lvmu
	#TCP Connections: $tcpc ESTABLISHED
	#User log       : $ulog
	#Network        : IP $ip ($mac)
	#Sudo           : $cmds cmd"
```

## AI Usage Disclosure

In compliance with 42's standards regarding the use of Artificial Intelligence:

* **Guided Learning & Troubleshooting:** **Gemini (Google AI)** was used as a technical consultant throughout the project. Specifically, it provided real-time troubleshooting during the Debian installation process when hardware compatibility issues or partitioning errors arose.
* **Concept Clarification:** AI assisted in breaking down complex system administration concepts, such as the UEFI/BIOS boot sequence, the PAM (Pluggable Authentication Modules) stack, and the security implications of LVM over LUKS.
* **Translation & Localization:** AI assisted in translating technical explanations from Spanish to English to maintain high-quality professional documentation.
* **Strict Policy:**
    * **No Blind Copy-Paste:** No scripts or configurations were directly copied. Gemini was used to explain the *logic* and *syntax*, while all implementation was performed manually by the author.
    * **Full Ownership:** Every line of code and configuration change is fully understood and can be replicated, modified, or explained in detail by the author during the defense.
    * **Verification:** AI-generated suggestions were always cross-referenced with official Debian documentation, man pages (`man sudoers`, `man sshd_config`), and the community resources listed below.


## Resources

### Official Documentation & Links
* **Debian OS:** [Official Download Site (NetInst)](https://www.debian.org/distrib/netinst) | [Why choose Debian?](https://www.debian.org/intro/why_debian.en.html)
* **VirtualBox:** [EFI Configuration - User Manual](https://docs.oracle.com/en/virtualization/virtualbox/6.0/user/efi.html)
* **Security:** [What is AppArmor? (AskUbuntu)](https://askubuntu.com/questions/236381/what-is-apparmor)

### Video Tutorials
* **Installation:** * [Install Debian 12/13 with Custom Partitions](https://www.youtube.com/watch?v=nmhSYKRnQsA)
    * [Rocky Linux Installation Guide](https://www.youtube.com/watch?v=30uePMhtcGc)
    * [Linux desde CERO - Nate Gentile](https://www.youtube.com/watch?v=knrc4q1S_q0&t=735s)
* **System Administration:**
    * [Domina SUDO en Linux - Etercloud](https://www.youtube.com/watch?v=Hc9Tfr3tY_A)
    * [LVM and Partitioning Deep Dive](https://www.youtube.com/watch?v=necFBdfls_8)

