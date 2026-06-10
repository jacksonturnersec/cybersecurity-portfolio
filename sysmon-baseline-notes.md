# Sysmon Baseline Notes

**Environment:** Windows 7 SP1 Lab (VirtualBox) | Sysmon v7.01  
**Date:** June 2026  
**Author:** Jackson Turner

---

## What is Sysmon

Sysmon (System Monitor) is an advanced, free Windows system service and device driver developed by Microsoft. It provides deep, granular telemetry on system activity — such as process creations, network connections, and file modifications — to help security teams track malware and investigate security incidents.

---

## Why SOC Analysts Use It

Standard Windows event logs are limited. Sysmon fills the gap by providing:

- Full command-line arguments and parent process for every execution — exposing suspicious scripts and malicious payloads
- Precise tracking of file creation, modification, and overwrite events, including registry changes
- Network connection telemetry that pairs outbound/inbound traffic with the exact executable responsible
- A foundation for SIEM correlation rules — Sysmon events feed directly into platforms like Wazuh, Splunk, and Microsoft Sentinel
- Detection coverage for advanced techniques including process injection, credential dumping, and fileless malware persistence

Sysmon is highly customisable. SOC teams frequently tune it using community-driven configuration files (such as SwiftOnSecurity's sysmonconfig) to filter background noise and log only relevant data — making it a powerful, free alternative to costly EDR agents.

---

## Installation

### Step 1 — Download
Download Sysmon from the Microsoft Sysinternals page and extract the `.zip` to a folder (e.g. `C:\Sysmon`).  
URL: https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon

### Step 2 — Install via Command Line

Open **Command Prompt as Administrator** (Start → type `cmd` → right-click → Run as administrator), then navigate to the Sysmon folder:

```
cd C:\Sysmon
```

Install with default configuration (no config file):

```
sysmon64.exe -accepteula -i
```

For 32-bit systems, use `sysmon.exe` instead of `sysmon64.exe`.

With a custom config file:

```
sysmon64.exe -accepteula -i sysmonconfig-export.xml
```

Expected output on success:
```
Sysmon installed.
SysmonDrv installed.
Starting SysmonDrv.
SysmonDrv started.
Starting Sysmon..
Sysmon started.
```

### Step 3 — Verify

Confirm the service is running:

```
sc query sysmon
```

Expected: `STATE: 4 RUNNING`

Open **Event Viewer** and navigate to:  
`Applications and Services Logs → Microsoft → Windows → Sysmon → Operational`

Events should begin populating immediately after installation.

---

## Key Event IDs

### Process & Execution

| Event ID | Name | Description |
|---|---|---|
| **1** | Process Create | Logs when a new process starts. Essential for command-line inspection and execution chain analysis. |
| **5** | Process Terminated | Records when a process exits. Useful for completing timelines and tracking malware cleanup. |
| **8** | CreateRemoteThread | Logs when a process creates a thread in another process — a classic code injection technique. |
| **25** | Process Tampering | Detects manipulation techniques like process hollowing or process herpaderping. |

### Network & Communication

| Event ID | Name | Description |
|---|---|---|
| **3** | Network Connection | Tracks TCP/UDP connections initiated by processes — identifies Command and Control (C2) traffic. |
| **22** | DNS Query | Logs DNS lookups issued by processes — early indicators of malicious infrastructure even when traffic is encrypted. |

### File System

| Event ID | Name | Description |
|---|---|---|
| **2** | File Creation Time Change | Records when a process modifies a file's creation timestamp — a common anti-forensic technique. |
| **11** | File Create | Logs when a file is created or overwritten. Essential for detecting dropped payloads and analysing temp folders. |
| **15** | File Create Stream Hash | Captures the hash of named file streams — threat actors use Alternate Data Streams to hide malicious code. |
| **23 & 26** | File Delete | Logs file deletion events. Event ID 23 also archives the deleted file. |

### Registry Activity

| Event ID | Name | Description |
|---|---|---|
| **12** | Registry Object Create/Delete | Logs addition or deletion of registry keys and values. |
| **13** | Registry Value Set | Captures changes to registry values — heavily utilised in persistence mechanisms. |
| **14** | Registry Key/Value Rename | Tracks when registry keys are renamed to obfuscate malicious configurations. |

### Advanced Threat & Injection

| Event ID | Name | Description |
|---|---|---|
| **7** | Image Load | Tracks when DLLs or drivers are loaded — useful for detecting DLL sideloading or injection. |
| **10** | Process Access | Records when one process accesses another — monitored to detect credential theft (e.g. targeting `lsass.exe`). |
| **19, 20, 21** | WMI Events | Tracks WMI filter, consumer, and binding registrations — common vectors for fileless malware persistence. |

### Service & Configuration

| Event ID | Name | Description |
|---|---|---|
| **4** | Sysmon Service State | Records when the Sysmon logging service stops or restarts. |
| **16** | Configuration Change | Logs when active filtering rules are updated — provides auditability of the monitoring configuration itself. |

---

## Sample Event — Process Create (Event ID 1)

The following event was captured from a live Sysmon installation on a Windows 7 SP1 lab environment immediately after deployment.

```
Process Create:
UtcTime: 2026-06-10 07:20:43.666
ProcessGuid: {7901ebac-104b-6a29-0000-001002920500}
ProcessId: 1040
Image: C:\Windows\System32\dllhost.exe
FileVersion: 6.1.7600.16385 (win7_rtm.090713-1255)
Description: COM Surrogate
Company: Microsoft Corporation
CommandLine: C:\Windows\system32\DllHost.exe /Processid:{E10F6C3A-F1AE-4ADC-AA9D-2FE65525666E}
CurrentDirectory: C:\Windows\system32\
User: NT AUTHORITY\SYSTEM
IntegrityLevel: System
Hashes: SHA1=ACE762C51DB1908C858C898D7E0F9B36F788D2D9
ParentProcessId: 576
ParentImage: C:\Windows\System32\svchost.exe
ParentCommandLine: C:\Windows\system32\svchost.exe -k DcomLaunch
```

**Analyst Assessment: Legitimate**

This event shows `dllhost.exe` (COM Surrogate) being spawned by `svchost.exe -k DcomLaunch` — the DCOM Launch service. This is expected Windows background behaviour. The process runs as `NT AUTHORITY\SYSTEM` with System integrity, consistent with an OS-initiated operation.

**What would make this suspicious:**
- `dllhost.exe` spawned by a user application like `winword.exe` or `chrome.exe`
- `CommandLine` pointing to a directory outside `C:\Windows\System32\`
- `IntegrityLevel` of High or System on a process launched from a user context
- Hash returning a positive result on VirusTotal

**The key analyst habit: always check the parent.** Malware frequently masquerades as legitimate executables. The image name alone is not enough — trace the full execution chain.

---

## Field Reference

| Field | What It Contains | Why Analysts Care |
|---|---|---|
| **UtcTime** | Timestamp in UTC | Correlate events across systems — always use UTC, never local time |
| **ProcessGuid** | Unique ID for this process instance | Track the process across multiple event types (creation, network, termination) |
| **ProcessId** | Numeric process ID (PID) | Cross-reference with Task Manager or other log sources |
| **Image** | Full path to the executable | Malware often uses legitimate names in wrong directories (e.g. `svchost.exe` in `C:\Temp\`) |
| **CommandLine** | Full command with arguments | Reveals encoded payloads, suspicious flags, living-off-the-land techniques |
| **CurrentDirectory** | Working directory at launch | Unusual directories (Temp, AppData, Downloads) are red flags |
| **User** | Account that launched the process | SYSTEM-level processes launched from a user context warrant investigation |
| **IntegrityLevel** | Trust level of the process | System > High > Medium > Low — elevation without cause is suspicious |
| **Hashes** | SHA1 (and others if configured) | Submit to VirusTotal or match against threat intel feeds |
| **ParentImage** | Executable that spawned this process | The most important field — unexpected parent-child relationships indicate injection or exploitation |
| **ParentCommandLine** | How the parent process was called | Confirms whether the parent itself was running legitimately |

---

*Part of the Jackson Turner cybersecurity portfolio — github.com/jacksonturnersec*
