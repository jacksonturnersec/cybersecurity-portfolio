# Windows & Linux Fundamentals — SOC Analyst Notes

**Author:** Jackson Turner | github.com/jacksonturnersec  
**Last updated:** June 2026  
**Source:** TryHackMe Pre-Security Path — Windows Fundamentals 1–3, Linux Fundamentals 1–3

---

## Table of Contents

1. [OS Fundamentals](#os-fundamentals)
2. [Windows Fundamentals](#windows-fundamentals)
3. [Linux Fundamentals](#linux-fundamentals)

---

## OS Fundamentals

### Privilege Layers

| Layer | Description |
|---|---|
| **Kernel Space** | The privileged core of the OS. Direct, unrestricted access to CPU, memory, storage, and hardware. The kernel manages all hardware and system resources. |
| **User Space** | Where all standard applications run. Cannot access hardware directly — must request kernel action via system calls. |

Every time an application opens a file, plays audio, or connects to a network, it makes a system call requesting the kernel to act on its behalf. This separation is a foundational security boundary.

### Interface Types

**GUI (Graphical User Interface)** — Icon-based interaction. Abstracts the underlying system from the user. What most non-technical users interact with daily.

**CLI (Command-Line Interface)** — Text-based commands. Provides greater precision, speed, and control. Required for most analyst and administrative work.

**SOC relevance:** Analysts work in both environments. Servers and security tools often have no GUI — CLI proficiency is non-negotiable.

---

## Windows Fundamentals

### System Configuration (MSConfig)

Accessed via: `msconfig` in the Run dialog or search bar.  
Purpose: Advanced troubleshooting — primarily used to diagnose startup issues.

| Tab | Purpose |
|---|---|
| **General** | Select what devices and services load on boot. Options: Normal, Diagnostic, Selective. |
| **Boot** | Define boot options for the OS. |
| **Services** | Lists all configured services regardless of state (running or stopped). |
| **Startup** | Microsoft directs startup management to Task Manager — MSConfig is NOT a startup management program. |
| **Tools** | List of system utilities with brief descriptions. |

---

### Advanced System Settings

Accessed via: Search "View advanced system settings" → System Properties panel.

**Crash Dump Configuration**  
Found under: Advanced tab → Settings under Startup and Recovery.

Windows creates a crash dump file when a critical error occurs (e.g. Blue Screen of Death). The **Write debugging information** dropdown sets the dump type:

| Dump Type | Description |
|---|---|
| Automatic memory dump | Windows decides what to capture |
| Kernel memory dump | Captures kernel memory only |
| Small memory dump (256 KB) | Minimal — basic error info |
| Complete memory dump | Full contents of RAM |
| None | No dump created |

**SOC relevance:** Crash dumps are forensic artefacts. Analysts and administrators use them to understand what caused a system failure — relevant in incident response when a system has been deliberately crashed or exploited.

---

### Computer Management (compmgmt)

Three primary sections: **System Tools**, **Storage**, and **Services and Applications**.

#### Task Scheduler
Creates and manages automated tasks. Tasks can be triggered at:
- System boot or user log in/log off
- Specific scheduled times (e.g. every 5 minutes)

View configured tasks under **Task Scheduler Library**.

**SOC relevance:** Attackers commonly abuse scheduled tasks for persistence — malware configures a task to re-execute after reboot. Reviewing the Task Scheduler Library is a standard persistence check.

---

#### Event Viewer
Records events occurring on the system — functions as an audit trail for diagnosing problems and investigating system activity.

**Three-pane layout:**
1. Left — hierarchical tree of event log providers
2. Centre — overview and summary of events for selected provider
3. Right — actions pane

**Five event types:**

| Type | Meaning |
|---|---|
| Error | Significant problem — data loss or functionality failure |
| Warning | Not currently critical, but could indicate a future problem |
| Information | Successful operation of an application, driver, or service |
| Success Audit | Audited security access attempt succeeded |
| Failure Audit | Audited security access attempt failed |

**Standard logs (under Windows Logs):**

| Log | Contains |
|---|---|
| Application | Events logged by applications |
| Security | Login attempts, privilege use, resource access — **primary SOC log** |
| Setup | Events related to application setup |
| System | Events logged by Windows system components |
| Forwarded Events | Events collected from remote machines |

**SOC relevance:** The Security log is the most analyst-relevant. Failed logon events (Event ID 4625), privilege escalation, and account changes all appear here. Event Viewer is the manual equivalent of what a SIEM aggregates and alerts on at scale.

---

#### Shared Folders
Lists all shares and folders that remote users can connect to. Useful for identifying unintended network exposure.

---

#### Local Users and Groups
Manage user accounts and group memberships on the local machine. Relevant for reviewing account privileges and identifying unauthorised accounts.

---

#### Performance Monitor (perfmon)
Views performance data in real-time or from a log file. Used for troubleshooting performance issues on local or remote systems.

---

#### Device Manager
View and configure hardware attached to the computer. Allows enabling/disabling hardware devices.

---

#### Disk Management (under Storage)
Performs advanced storage tasks:
- Set up a new drive
- Extend or shrink a partition
- Assign or change a drive letter (e.g. E:)

---

#### Services and Applications
Lists services running on the system. Right-click any service → Properties to see:
- Service name
- Path to executable
- Startup type

**Startup type options:**

| Type | Behaviour |
|---|---|
| Automatic | Starts every time the system boots |
| Manual | Only starts when triggered by another process or user |
| Disabled | Should not run at all |

**SOC relevance:** Malware frequently installs itself as a service set to Automatic. Reviewing unexpected services with unusual executable paths is a standard triage step.

---

### Resource Monitor (resmon)

Displays per-process and aggregate resource usage. More granular than Task Manager.

**Overview tab — four sections:**

| Section | Shows |
|---|---|
| CPU | Per-process CPU usage |
| Memory | RAM allocation per process |
| Disk | Read/write activity per process |
| Network | Active network connections per process |

Additional capability: identify deadlocked processes and file locking conflicts. Can start, stop, pause, and resume services from the UI.

**SOC relevance:** Useful for identifying processes with unexpected network connections or unusual disk activity during triage.

---

### Command Prompt — Core Commands

| Command | Purpose |
|---|---|
| `hostname` | Outputs the computer name |
| `whoami` | Outputs the name of the logged-in user |
| `ipconfig` | Displays network address settings |
| `ipconfig /?` | Opens help manual for ipconfig |
| `cls` | Clears the command prompt screen |
| `netstat` | Displays protocol statistics and current TCP/IP connections |
| `net` | Manages network resources — use `net help` for sub-command list |
| `command /?` | Retrieves help manual for most commands |

**SOC relevance:** These are standard initial triage commands on a Windows host. `whoami`, `ipconfig`, and `netstat` are typically run early in host investigation to establish context.

---

### Registry Editor

A central hierarchical database used to store configuration information for users, applications, and hardware.

The registry is continuously referenced by Windows during operation. It stores:
- User profiles
- Installed applications and their associated file types
- Folder and icon settings
- Hardware present on the system
- Ports in use

**SOC relevance:** The registry is a primary persistence location. Attackers commonly add entries to `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run` (or similar Run keys) to execute malware at startup. Registry analysis is a core skill in malware investigation and incident response.

---

### Windows Updates — Patch Tuesday

Windows security updates are typically released on the **second Tuesday of each month** — known as **Patch Tuesday**.

Critical vulnerabilities may trigger out-of-band updates pushed immediately via Windows Update.

**SOC relevance:** Unpatched systems are a primary attack surface. Patch compliance monitoring is a standard SOC and vulnerability management function.

---

### Windows Security

#### Virus & Threat Protection

**Scan types:**

| Scan | Scope |
|---|---|
| Quick scan | Common threat locations only |
| Full scan | All files and running programs — may take over an hour |
| Custom scan | User-defined files and locations |

**Threat history sections:**
- **Last scan** — result of most recent automated scan
- **Quarantined threats** — isolated, prevented from running, periodically removed
- **Allowed threats** — items flagged as threats but permitted by user

**Key protection settings:**

| Setting | Function |
|---|---|
| Real-time protection | Detects and blocks malware attempting to install or run |
| Cloud-delivered protection | Faster updates via cloud threat intelligence |
| Automatic sample submission | Sends suspicious files to Microsoft |
| Controlled folder access | Blocks unauthorised changes to protected folders |
| Exclusions | Files/folders excluded from scanning — **treat with caution** |

**SOC relevance:** Controlled folder access is a ransomware mitigation control. Finding unexpected exclusions on a host is a red flag — attackers add exclusions to prevent detection.

---

#### Firewall & Network Protection

Controls what traffic is allowed through ports.

**Three firewall profiles:**

| Profile | When it applies |
|---|---|
| **Domain** | Networks where the host can authenticate to a domain controller |
| **Private** | User-assigned — home or trusted private networks |
| **Public** | Default — public networks (cafes, airports, hotspots) |

Each profile can be configured to: turn firewall on/off, or block all incoming connections.

**SOC relevance:** Firewall profiles affect which rules are active. Attackers may attempt to disable the firewall or add inbound exceptions to maintain access. Reviewing firewall state and rules is part of standard host hardening and incident triage.

---

#### App & Browser Control — Microsoft Defender SmartScreen

Protects against:
- Phishing and malware websites
- Malicious applications
- Potentially malicious file downloads

---

### TPM — Trusted Platform Module

Hardware-based security technology. A TPM chip is a secure crypto-processor designed to:
- Perform cryptographic operations
- Resist physical tampering
- Prevent malicious software from interfering with security functions

---

### BitLocker

Data protection feature integrated with the OS. Protects against data theft or exposure from lost, stolen, or decommissioned devices.

Best protection achieved when used with a TPM chip.

**SOC relevance:** BitLocker is a data-at-rest encryption control. Its presence or absence is relevant in asset management and in assessing the impact of a device being physically compromised.

---

## Linux Fundamentals

### Core Commands

| Command | Function |
|---|---|
| `echo` | Outputs text (e.g. `echo "Hello World"`) |
| `whoami` | Displays currently logged-in user |
| `ls` | Lists files and directories |
| `cd` | Changes directory |
| `pwd` | Prints working directory (full path) |
| `cat` | Outputs file contents (e.g. `cat todo.txt`) |
| `find` | Locates files (e.g. `find -name *.txt`) |
| `grep` | Searches file contents for a value (e.g. `grep "192.168.1.1" access.log`) |
| `grep -R` | Recursive search across all files in a directory |
| `touch` | Creates a file |
| `mkdir` | Creates a directory |
| `cp` | Copies a file or folder |
| `mv` | Moves a file or folder |
| `rm` | Removes a file or folder |
| `file` | Determines the type of a file |
| `su` | Switches user |
| `man` | Opens the manual page for a command (e.g. `man ls`) |

---

### Operators

| Operator | Function |
|---|---|
| `&` | Runs a command in the background |
| `&&` | Chains commands — second command runs only if first succeeds |
| `>` | Redirects output to a file — overwrites existing content |
| `>>` | Redirects output to a file — appends, does not overwrite |

---

### SSH — Secure Shell

Protocol for encrypted communication between devices. Input is encrypted in transit and decrypted on arrival at the remote machine. Used for secure remote login and remote command execution.

**Connect:** `ssh username@MACHINE_IP`

---

### File Permissions

Linux permissions are expressed as a three-part numeric value for owner, group, and others:

| Value | Permission |
|---|---|
| 4 | Read (r) |
| 2 | Write (w) |
| 1 | Execute (x) |

Combine values to set permissions (e.g. 7 = read + write + execute, 6 = read + write, 5 = read + execute).

**SOC relevance:** Misconfigured permissions are a common attack path. Identifying world-writable files or SUID binaries is part of privilege escalation analysis on Linux systems.

---

### Common Directories

| Directory | Purpose |
|---|---|
| `/etc` | System configuration files used by the OS |
| `/var` | Variable data — logs, databases, application data |
| `/root` | Home directory for the root user (not `/home/root`) |
| `/tmp` | Temporary files — cleared on reboot. Volatile storage. |

**SOC relevance:** `/tmp` is frequently abused by attackers as a staging area for malware — it is writable by all users and cleared on reboot, making it useful for hiding temporary payloads. `/var/log` contains system and application logs.

---

### Text Editors

**Nano** — Simple, beginner-friendly terminal text editor. Create or edit files directly from the command line.

**Vim** — Advanced editor with customisable shortcuts, syntax highlighting, and broad terminal compatibility. Works across environments where nano may not be installed.

---

### File Transfers

**Downloading files from the web:**
```
wget https://example.com/file.txt
```

**Serving files from a host using Python3:**
```
python3 -m http.server
```
This starts a web server on port 8000. Download from it using:
```
wget http://MACHINE_IP:8000/filename
```
Run wget in a separate terminal tab — do not close the server terminal.

---

### Process Management

Processes are programs running on the machine, managed by the kernel.

**View running processes:**
```
ps aux
```

**Kill signals:**

| Signal | Behaviour |
|---|---|
| SIGTERM | Terminates process — allows cleanup first |
| SIGKILL | Force-kills immediately — no cleanup |
| SIGSTOP | Suspends/pauses the process |

**Manage services on boot:**
```
systemctl [start|stop|enable|disable|status] servicename
```

**Bring a background process to the foreground:**
```
fg
```

---

### Cron Jobs — Scheduled Automation

The cron process (`cron`) starts at boot and manages scheduled task execution via **crontabs**.

**Edit crontab:**
```
crontab -e
```

**Crontab field order:**

| Field | Value |
|---|---|
| MIN | Minute (0–59) |
| HOUR | Hour (0–23) |
| DOM | Day of month (1–31) |
| MON | Month (1–12) |
| DOW | Day of week (0–7, 0 and 7 = Sunday) |
| CMD | Command to execute |

**Special syntax:** `@reboot` runs the command once at system startup.

**SOC relevance:** Cron is a common Linux persistence mechanism. Attackers add cron entries to re-execute malicious commands on schedule or at reboot. Reviewing crontab entries is a standard persistence check on Linux hosts.

---

### Log Files

**Key log locations:**

| Path | Contents |
|---|---|
| `/var/log/syslog` | General system activity |
| `/var/log/auth.log` | Authentication events — logins, sudo usage |
| `/var/log/apache2/access.log` | Web server access log |
| `/var/log/apache2/error.log` | Web server error log |

**SOC relevance:** Log files are the primary data source for Linux-based investigation. `auth.log` is particularly important — it records all SSH logins, failed authentication attempts, and privilege escalation via sudo. Searching logs with `grep` is a fundamental analyst skill.

---

*These notes document core Windows and Linux concepts as they apply to security operations work. Each tool and command listed has direct relevance to endpoint analysis, incident triage, or threat detection in a SOC environment.*
