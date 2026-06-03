# Windows & Linux Fundamentals
**Author:** Jackson Turner | [github.com/jacksonturnersec](https://github.com/jacksonturnersec)  
**Last updated:** June 2026  
**Source:** TryHackMe Pre-Security Path — Linux Fundamentals (Parts 1–3), OS Introduction, Windows Basics  
**Relevance:** Core knowledge for SOC Analyst roles — understanding OS behaviour, file systems, process management, and automation is essential for log analysis and incident triage.

---

## Table of Contents
1. [Operating Systems — Core Concepts](#1-operating-systems--core-concepts)
2. [Windows Basics](#2-windows-basics)
3. [Linux Fundamentals](#3-linux-fundamentals)
   - [Core Commands](#core-commands)
   - [Operators](#operators)
   - [SSH](#ssh)
   - [File Permissions](#file-permissions)
   - [Common Directories](#common-directories)
   - [Text Editors](#text-editors)
   - [File Transfers](#file-transfers)
   - [Process Management](#process-management)
   - [Automation — Cron Jobs](#automation--cron-jobs)
   - [Log Files](#log-files)

---

## 1. Operating Systems — Core Concepts

### Privilege Layers

| Layer | Description |
|---|---|
| **Kernel Space** | Privileged core of the OS. The kernel manages hardware directly — CPU, memory, storage. Has unrestricted access. |
| **User Space** | Where all standard applications run. Cannot access hardware directly. Must make a **system call** to request the kernel act on its behalf. |

> **SOC relevance:** Malware that escapes User Space and operates in Kernel Space (rootkits) is far more dangerous and harder to detect. Privilege escalation attacks aim to move from User Space to Kernel Space.

### GUI vs CLI

| Interface | Description |
|---|---|
| **GUI (Graphical User Interface)** | Visual, icon-based navigation. What most end users interact with. Limited precision for advanced tasks. |
| **CLI (Command-Line Interface)** | Text-based commands. Offers greater precision, speed, and control. Essential for analysts working in Linux environments. |

### Core OS Functions
At a basic level, every operating system handles:
- **Authentication** — verifying who you are
- **Permissions** — controlling what you can access
- **Isolation** — keeping processes separate from each other
- **System Protection** — preventing unauthorised access to core resources

---

## 2. Windows Basics

> **Note:** This section will be expanded when the TryHackMe Windows Fundamentals module is completed. Current coverage is from the OS Introduction room.

Windows is the dominant OS in enterprise environments. From a SOC perspective, understanding Windows is critical because:
- Most endpoint logs (Event Logs, Sysmon) come from Windows machines
- Active Directory runs on Windows Server
- The majority of malware targets Windows

*Full Windows Fundamentals notes — coming in Week 3.*

---

## 3. Linux Fundamentals

Linux underpins most servers, security tools, and cloud infrastructure. As a SOC analyst, you will work with Linux constantly — reading logs, running commands, managing files, and interacting with security tooling.

---

### Core Commands

| Command | Function | Example |
|---|---|---|
| `echo` | Outputs text to the terminal | `echo "Hello World"` |
| `whoami` | Shows the current logged-in user | `whoami` |
| `ls` | Lists files and directories | `ls` |
| `cd` | Changes directory | `cd /var/log` |
| `pwd` | Prints the current working directory | `pwd` |
| `cat` | Outputs the contents of a file | `cat access.log` |
| `find` | Locates files by name or type | `find -name *.txt` |
| `grep` | Searches file contents for a specific value | `grep "192.168.1.1" access.log` |
| `man` | Opens the manual page for a command | `man ls` |
| `touch` | Creates a new file | `touch notes.txt` |
| `mkdir` | Creates a new directory | `mkdir logs` |
| `cp` | Copies a file or folder | `cp file.txt /backup/` |
| `mv` | Moves or renames a file or folder | `mv old.txt new.txt` |
| `rm` | Removes a file or folder | `rm file.txt` |
| `file` | Determines the file type | `file suspicious.bin` |
| `su` | Switches to another user | `su root` |

> **Grep tip:** To search recursively across all files in a directory:
> ```
> grep -R "SEARCH_TERM" /path/to/directory/
> ```

---

### Operators

| Operator | Function |
|---|---|
| `&` | Runs a command in the background |
| `&&` | Chains two commands — second only runs if first succeeds |
| `>` | Redirects output to a file (overwrites if file exists) |
| `>>` | Redirects output to a file (appends — does not overwrite) |

**Example:**
```bash
echo "hello" > welcome.txt    # Creates/overwrites welcome.txt with "hello"
echo "world" >> welcome.txt   # Appends "world" to welcome.txt
```

---

### SSH

**Secure Shell (SSH)** is an encrypted protocol used to remotely access and manage Linux machines.

- All input/output is encrypted in transit using cryptography
- Human-readable input is encrypted before travelling across the network
- Decrypted once it reaches the remote machine
- Runs on **port 22**

```bash
ssh username@10.10.10.10
```

> **SOC relevance:** SSH sessions appear in auth logs (`/var/log/auth.log`). Brute-force SSH attacks are common. Seeing multiple failed SSH attempts followed by a success is a key alert pattern.

---

### File Permissions

Linux uses numeric (octal) values to define file permissions across three groups: **Owner**, **Group**, and **Others**.

| Permission | Numeric Value |
|---|---|
| Read (r) | 4 |
| Write (w) | 2 |
| Execute (x) | 1 |

Permissions are combined: `rwx = 7`, `rw- = 6`, `r-- = 4`

**Example:** `chmod 755 script.sh` gives the owner full access, and group/others read + execute.

> **Why this matters for SOC:** Identifying files with unusual permissions (e.g. world-writable scripts, SUID binaries) is a core part of Linux-based incident investigation.

---

### Common Directories

| Directory | Purpose |
|---|---|
| `/etc` | System configuration files |
| `/var` | Variable data — logs, databases, application data |
| `/var/log` | Log files (apache2, auth, syslog, etc.) |
| `/root` | Home directory for the root user |
| `/tmp` | Temporary files — cleared on reboot |
| `/home` | Home directories for standard users |

> **SOC relevance:** Attackers frequently drop files in `/tmp` because it is writable by all users and cleared on reboot — making it useful for staging malware that doesn't need to persist.

---

### Text Editors

| Editor | Notes |
|---|---|
| `nano` | Simple, beginner-friendly. Open with `nano filename.txt`. Exit with `Ctrl+X`. |
| `vim` | Advanced editor. Supports syntax highlighting, custom keybindings. Available across more environments than nano. |

---

### File Transfers

**Downloading files with wget:**
```bash
wget https://example.com/file.txt
```

**Serving files via Python HTTP server:**
```bash
# On the host machine — starts a web server on port 8000
python3 -m http.server

# On the receiving machine — download the file
wget http://MACHINE_IP:8000/filename.txt
```

> Always open a new terminal tab before running `wget` — the python3 server must keep running in the original tab.

---

### Process Management

Processes are programs running on the machine, managed by the kernel.

**Useful commands:**

| Command | Function |
|---|---|
| `ps aux` | Lists all running processes |
| `top` | Real-time process viewer |
| `kill [PID]` | Sends a signal to a process |

**Kill signals:**

| Signal | Behaviour |
|---|---|
| `SIGTERM` | Terminate cleanly — allows the process to do cleanup first |
| `SIGKILL` | Force kill — no cleanup |
| `SIGSTOP` | Suspend/pause the process |

**Managing services at boot with systemctl:**

```bash
systemctl start myservice      # Start a service
systemctl stop myservice       # Stop a service
systemctl enable myservice     # Start automatically at boot
systemctl disable myservice    # Disable from starting at boot
systemctl status myservice     # Check current status
```

> Process IDs (PIDs) increment sequentially. If the last process ID was 300, the next will be 301.

---

### Automation — Cron Jobs

The **cron** process allows users to schedule tasks to run automatically at defined times. Cron jobs are configured using **crontabs**.

```bash
crontab -e    # Edit your crontab
```

**Crontab syntax — 6 required values:**

```
MIN  HOUR  DOM  MON  DOW  CMD
```

| Field | Meaning |
|---|---|
| MIN | Minute (0–59) |
| HOUR | Hour (0–23) |
| DOM | Day of the month (1–31) |
| MON | Month (1–12) |
| DOW | Day of the week (0–7, where 0 and 7 = Sunday) |
| CMD | The command to run |

**Special strings:**

| String | Meaning |
|---|---|
| `@reboot` | Run once at system startup |
| `@daily` | Run once per day |
| `@hourly` | Run once per hour |

**Example:**
```bash
0 */12 * * * cp -R /home/user/Documents /var/backups/ >/dev/null 2>&1
```
This runs a backup every 12 hours, suppressing output.

> **SOC relevance:** Attackers use cron jobs for persistence — scheduling malicious scripts to run at reboot or on a regular interval. Reviewing crontabs is a standard step in Linux-based incident response.

---

### Log Files

Linux stores log files in `/var/log`. Two key Apache2 log types:

| Log | Purpose |
|---|---|
| `access.log` | Records every HTTP request made to the server — IP address, file accessed, status code, timestamp |
| `error.log` | Records errors encountered by the web server |

**Reading a log file:**
```bash
cat /var/log/apache2/access.log
grep "192.168.1.1" /var/log/apache2/access.log    # Filter by IP
```

**Access log format:**
```
IP_ADDRESS - - [TIMESTAMP] "METHOD /file HTTP/version" STATUS_CODE BYTES
```

> **SOC relevance:** Access logs are a primary source of evidence during web-based incident investigations. You will use `grep` to filter by IP, file path, or status code to identify malicious requests.

---

*Part of the [cyber-fast-track](https://github.com/jacksonturnersec) portfolio — building toward SOC Analyst roles in Melbourne, Australia.*
