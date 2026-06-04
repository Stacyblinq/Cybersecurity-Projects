# Essential Windows and PowerShell Skills for SOC Analysts

## Project Overview

This project simulates a real-world Security Operations Center (SOC) investigation involving brute-force attacks, credential theft, Pass-the-Hash attacks, lateral movement, malicious PowerShell execution, persistence mechanisms, credential dumping, and Command-and-Control (C2) communication.

Using Windows Security Logs, Sysmon Logs, and PowerShell Operational Logs, I investigated attacker activities, correlated events across multiple log sources, and reconstructed the attack timeline to identify the attacker's tactics, techniques, and procedures (TTPs).

---

## Project Details

| Category | Information |
|-----------|------------|
| Project Title | Essential Windows and PowerShell Skills for SOC Analysts |
| Environment | Windows 10 Virtual Machine |
| Log Sources | Security.csv, Sysmon.csv, PowerShell.csv |
| Focus Area | Threat Hunting, Incident Response, Log Analysis |
| Analyst | Ebere Anastasia Akpan |

---

## Objectives

- Analyze Windows Security Event Logs.
- Detect brute-force attacks.
- Identify Pass-the-Hash activity.
- Investigate lateral movement techniques.
- Hunt for malicious processes using Sysmon.
- Detect PowerShell abuse and persistence mechanisms.
- Correlate multiple log sources to reconstruct attack timelines.
- Apply incident response methodologies.

---

## Tools & Technologies

### Operating Systems

- Windows 10

### Windows Security Tools

- Windows Event Logs
- Sysmon
- PowerShell

### Security Concepts

- Threat Hunting
- Incident Response
- Event Correlation
- Attack Timeline Reconstruction
- Credential Theft Detection
- Lateral Movement Analysis

### Attack Techniques Investigated

- Brute Force Attacks
- Pass-the-Hash
- Credential Dumping
- PowerShell Abuse
- Persistence Mechanisms
- Command and Control (C2)
- Lateral Movement
- Defense Evasion

---

# Investigation Methodology

## Part 1: Windows Security Event Log Analysis

### Detecting Brute-Force Attacks

Windows Event ID:

```text
4625
```

Event ID 4625 represents failed logon attempts.

### PowerShell Query

```powershell
$securityLogs = Import-Csv .\Security.csv

$failedLogons = $securityLogs | Where-Object {
    $_.EventID -eq '4625'
}

$failedLogons | Group-Object SubjectUserName |
Sort-Object Count -Descending |
Select-Object Count, Name

$failedLogons | Group-Object IpAddress |
Sort-Object Count -Descending |
Select-Object Count, Name
```

### Findings

Targeted Account:

```text
administrator
```

Attacker Source IP:

```text
10.10.50.99
```

Failed Logon Attempts:

```text
150
```

### Analysis

The logs revealed a brute-force attack against the administrator account originating from a single source IP address.

---

## Detecting Pass-the-Hash Activity

Windows Event ID:

```text
4624
```

Logon Type:

```text
9 (NewCredentials)
```

### PowerShell Query

```powershell
$securityLogs |
Where-Object {
    $_.EventID -eq '4624' -and
    $_.LogonType -eq '9'
} |
Select-Object TimeCreated, Computer,
SubjectUserName, IpAddress
```

### Findings

Source IP:

```text
10.10.50.99
```

Lateral Movement Targets:

```text
SRV-FILE02
SRV-SQL04
WS-FINANCE01
WS-MGMT04
```

### Analysis

The attacker successfully used Pass-the-Hash techniques to authenticate to multiple systems using stolen credentials.

---

# Part 2: Sysmon Threat Hunting

## Detecting Ransomware Preparation Activity

Sysmon Event ID:

```text
1
```

### PowerShell Query

```powershell
$sysmonLogs = Import-Csv .\Sysmon.csv

$sysmonLogs |
Where-Object {
    $_.EventID -eq '1' -and
    $_.CommandLine -match
    'vssadmin.*delete shadows'
} |
Select-Object TimeCreated,
Computer, User, CommandLine
```

### Findings

Suspicious Command:

```cmd
vssadmin.exe delete shadows /all /quiet
```

Affected Host:

```text
WS-DC01
```

### Analysis

The command deletes Volume Shadow Copies, a technique commonly used by ransomware operators to prevent recovery of encrypted files.

---

## Detecting C2 Communication

Sysmon Event ID:

```text
3
```

### PowerShell Query

```powershell
$sysmonLogs |
Where-Object {
    $_.EventID -eq '3' -and
    $_.Process -match 'powershell.exe'
} |
Group-Object DestinationIP |
Sort-Object Count -Descending |
Select-Object Count, Name
```

### Findings

Suspected C2 Server:

```text
185.220.101.45
```

### Analysis

PowerShell initiated outbound communications to an external IP address likely functioning as a Command-and-Control server.

---

## Detecting Credential Dumping

Sysmon Event ID:

```text
10
```

### PowerShell Query

```powershell
$sysmonLogs |
Where-Object {
    $_.EventID -eq '10' -and
    $_.TargetProcess -match 'lsass.exe'
} |
Select-Object TimeCreated,
Process,
TargetProcess,
Description
```

### Findings

Malicious Process:

```text
mimi.exe
```

Target Process:

```text
lsass.exe
```

### Analysis

The attacker accessed LSASS memory to dump credentials, a common Mimikatz technique used to steal passwords and authentication tokens.

---

# Part 3: PowerShell Threat Analysis

## Detecting Obfuscated PowerShell Commands

PowerShell Event ID:

```text
4104
```

### PowerShell Query

```powershell
$psLogs = Import-Csv .\PowerShell.csv

$psLogs |
Where-Object {
    $_.EventID -eq '4104' -and
    $_.ScriptBlockText -match
    '-Enc|-EncodedCommand'
} |
Select-Object TimeCreated,
ScriptBlockText
```

### Findings

Observed Indicators:

```text
-EncodedCommand
-NoP
-W Hidden
```

### Analysis

The attacker used encoded PowerShell commands and hidden execution flags to evade detection.

---

## Detecting Persistence

### Findings

The attacker created a scheduled task using:

```powershell
Register-ScheduledTask
```

and

```powershell
New-ScheduledTaskAction
```

Startup Trigger:

```powershell
-AtStartup
```

### Analysis

This persistence mechanism ensured malicious code would execute automatically after system reboot.

---

## Decoding Malicious PowerShell Payloads

### Findings

Decoded Payload Functionality:

```text
Reverse Shell
```

Destination:

```text
135.220.102.45
```

Port:

```text
4444
```

### Analysis

The decoded command attempted to establish a reverse shell connection, allowing the attacker to remotely control the compromised system.

---

# Incident Response Investigation

## Initial Access

### Timeline

First Successful Authentication:

```text
2026-05-14 08:13:05
```

### Analysis

The brute-force attack succeeded shortly before lateral movement activities began.

---

## Privilege Escalation

### Findings

The attacker gained access to:

```text
administrator
```

credentials and leveraged:

```text
Logon Type 9 (NewCredentials)
```

to authenticate across multiple systems.

---

## Persistence

### Technique

```text
Scheduled Task Creation
```

### Indicators

```powershell
Register-ScheduledTask
New-ScheduledTaskAction
```

### Purpose

Maintain access after reboot.

---

## Lateral Movement

### Tool Identified

```text
PsExec.exe
```

### Target Host

```text
WS-MGMT04
```

### Analysis

The attacker remotely executed commands on additional systems to expand access within the environment.

---

# Attack Timeline

| Phase | Activity |
|---------|----------|
| Initial Access | Brute-force attack against administrator account |
| Credential Access | Successful authentication from 10.10.50.99 |
| Lateral Movement | Pass-the-Hash across multiple systems |
| Execution | Malicious PowerShell execution |
| Persistence | Scheduled task creation |
| Credential Dumping | mimi.exe accessed lsass.exe |
| Defense Evasion | vssadmin shadow copy deletion |
| Command & Control | Connection to external C2 server |
| Lateral Movement | PsExec execution on WS-MGMT04 |

---

# MITRE ATT&CK Mapping

| Tactic | Technique |
|----------|-----------|
| Credential Access | T1110 - Brute Force |
| Credential Access | T1003.001 - LSASS Memory |
| Lateral Movement | T1550.002 - Pass the Hash |
| Lateral Movement | T1021 - Remote Services |
| Execution | T1059.001 - PowerShell |
| Persistence | T1053.005 - Scheduled Task |
| Defense Evasion | T1070.004 - File Deletion |
| Command and Control | T1071 - Application Layer Protocol |

---

# Recommendations

## Authentication Security

- Implement Multi-Factor Authentication (MFA).
- Enforce strong password policies.
- Monitor failed logon attempts.

## Endpoint Security

- Deploy Endpoint Detection and Response (EDR).
- Enable Sysmon monitoring.
- Restrict administrative privileges.

## PowerShell Security

- Enable Script Block Logging.
- Monitor encoded PowerShell commands.
- Restrict unnecessary PowerShell execution.

## Network Security

- Implement network segmentation.
- Monitor lateral movement indicators.
- Restrict administrative remote execution tools.

---

# Skills Demonstrated

## SOC Operations

- Security Monitoring
- Incident Investigation
- Event Correlation
- Threat Hunting
- Incident Response
- Attack Timeline Reconstruction

## Windows Security

- Windows Event Log Analysis
- Sysmon Analysis
- PowerShell Analysis
- Threat Detection

## Threat Detection

- Brute Force Detection
- Pass-the-Hash Investigation
- Credential Dumping Detection
- Persistence Analysis
- C2 Detection
- Lateral Movement Investigation

---

# Lessons Learned

This project strengthened my ability to:

- Analyze Windows Security Logs.
- Investigate Sysmon telemetry.
- Detect advanced attacker techniques.
- Correlate multiple data sources.
- Reconstruct attack timelines.
- Investigate PowerShell-based attacks.
- Identify persistence mechanisms.
- Respond to enterprise security incidents.

---

# Screenshots

Create an `/images` folder and add screenshots from the lab.

Example:

```markdown
![Brute Force Detection](images/bruteforce-detection.png)

![Pass the Hash Activity](images/pass-the-hash.png)

![Credential Dumping](images/credential-dumping.png)

![PowerShell Persistence](images/powershell-persistence.png)

![Attack Timeline Investigation](images/attack-timeline.png)
```

---

# Project Documentation

Full project report included in this repository.

```text
Essential_Windows_and_PowerShell_Skills_for_SOC_Analysts.pdf
```

---

# Author

**Ebere Anastasia Akpan**

Cybersecurity Analyst | SOC Analyst | Cloud Security (AWS) | IT Support

## Connect With Me

- LinkedIn: https://linkedin.com/in/YOUR-LINKEDIN
- GitHub: https://github.com/YOUR-USERNAME
