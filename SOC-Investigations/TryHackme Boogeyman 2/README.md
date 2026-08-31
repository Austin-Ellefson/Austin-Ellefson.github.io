# TryHackMe — Boogeyman 2

## Phishing & Memory Forensics Investigation

> **Platform:** TryHackMe  
> **Room:** Boogeyman 2  
> **Focus:** Phishing Analysis, Malicious Office Documents, Memory Forensics, Incident Response  
> **Primary Tools:** Volatility 3, olevba, strings, grep

---

## Overview

Boogeyman 2 continues the investigation into the Boogeyman threat actor, with the analysis focusing heavily on **phishing and Windows memory forensics**.

The incident begins when **Maxine Beck**, an HR employee at Quick Logistics LLC, receives what appears to be a legitimate job application. The attached Microsoft Word resume is malicious.

My goal was to reconstruct the compromise from the original phishing email through malicious document execution, staged payload delivery, command-and-control (C2) communication, and persistence.

The primary evidence consisted of the original phishing email, malicious Word attachment, and a memory dump from the compromised workstation.

For some Volatility portions, I referenced a Boogeyman 2 walkthrough to help identify useful plugins and approaches. I used those suggestions as starting points, then examined the resulting artifacts to understand what each established in the attack chain.

---

## Investigation

### 1. Initial Phishing Investigation

I started with the phishing email to establish the initial access vector.

**Sender:** `westaylor23@outlook.com`  
**Target:** `maxine.beck@quicklogisticsorg.onmicrosoft.com`  
**Attachment:** `Resume_WesleyTaylor.doc`

The lure made sense from a social-engineering perspective: an HR employee receiving resumes is expected business activity.

**MITRE ATT&CK:** `T1566.001 — Spearphishing Attachment`

### 2. Fingerprinting the Malicious Document

I calculated the attachment's MD5 hash:

```text
52c4384a0b9e248b95804352ebec6c5b
```

Hashing gave me a reproducible identifier that could be used for threat-intelligence searches and hunting across EDR, SIEM, and email-security telemetry.

### 3. Analyzing the Word Macro

I used `olevba` to inspect the document for embedded VBA:

```bash
olevba Resume_WesleyTaylor.doc
```

The macro contacted:

```text
https://files.boogeymanisback[.]lol/aa2a9c53cbb80416d3b47d85538d9971/update.png
```

Despite the `.png` extension, the downloaded content was ultimately written to:

```text
C:\ProgramData\update.js
```

It was executed with `wscript.exe`.

```text
Phishing Email
      |
      v
Resume_WesleyTaylor.doc
      |
      v
Malicious VBA Macro
      |
      v
C:\ProgramData\update.js
      |
      v
wscript.exe
```

**MITRE ATT&CK:** `T1204.002`, `T1059.005`, `T1059.007`

### 4. Memory Forensics — Process Tree

I moved into the workstation memory image and examined process ancestry:

```bash
vol -f WKSTN-2961.raw windows.pstree
```

I identified `wscript.exe` with PID `4260`. Its parent was PID `1124`, `WINWORD.EXE`.

```text
WINWORD.EXE
     |
     v
wscript.exe (PID 4260)
```

This directly connected the malicious Word document to script execution. From a SOC perspective, an Office application spawning Windows Script Host is a strong behavioral lead.

### 5. Discovering the Next Payload

I searched the memory image for references to attacker infrastructure:

```bash
strings WKSTN-2961.raw | grep boogeymanisback
```

This revealed another payload:

```text
https://files.boogeymanisback[.]lol/aa2a9c53cbb80416d3b47d85538d9971/update.exe
```

The attack was therefore multi-stage:

```text
Resume_WesleyTaylor.doc
          |
          v
       VBA Macro
          |
          v
C:\ProgramData\update.js
          |
          v
      wscript.exe
          |
          v
       update.exe
```

### 6. Identifying C2 Traffic

I used Volatility to inspect network artifacts:

```bash
vol -f WKSTN-2961.raw windows.netscan
```

A suspicious connection was associated with PID `6216` and the remote endpoint:

```text
128.199.95.189:8080
```

This provided evidence of command-and-control communication.

**MITRE ATT&CK:** `T1071 — Application Layer Protocol`

### 7. Identifying the Malicious Executable

I investigated PID 6216 further:

```bash
vol -f WKSTN-2961.raw windows.dlllist --pid 6216
```

This revealed:

```text
C:\Windows\Tasks\updater.exe
```

I could now correlate the process, executable, and network connection:

```text
PID 6216
   |
   +--> C:\Windows\Tasks\updater.exe
   |
   +--> 128.199.95.189:8080
```

### 8. Recovering the Outlook Attachment Path

I searched memory for the original resume:

```bash
vol -f WKSTN-2961.raw windows.filescan | grep Resume_WesleyTaylor
```

This revealed:

```text
C:\Users\maxine.beck\AppData\Local\Microsoft\Windows\INetCache\Content.Outlook\WQHGZCFI\Resume_WesleyTaylor (002).doc
```

This connected the forensic evidence back to Outlook's cached attachment location.

### 9. Investigating Persistence

I dumped the suspicious process memory:

```bash
vol -f WKSTN-2961.raw windows.memmap --dump --pid 6216
```

Then searched it for scheduled-task activity:

```bash
strings pid.6216.dmp | grep -e 'schtasks'
```

I broadened the search to the full memory image:

```bash
strings WKSTN-2961.raw | grep schtasks
```

The attacker created a scheduled task named `Updater`, configured to execute daily at `09:00`.

```text
schtasks /Create /F /SC DAILY /ST 09:00 /TN Updater ...
```

The task launched hidden PowerShell and retrieved encoded content from:

```text
HKCU:\Software\Microsoft\Windows\CurrentVersion\debug
```

using the `debug` property. The data was Base64-decoded, converted from Unicode, and passed to `IEX` (`Invoke-Expression`).

```text
Scheduled Task: Updater
        |
        v
powershell.exe
        |
        v
Read Registry Value
        |
        v
Base64 Decode
        |
        v
Unicode Decode
        |
        v
Invoke-Expression
        |
        v
Execute Payload
```

**MITRE ATT&CK:** `T1053.005`, `T1112`, `T1059.001`, `T1027`

---

## Reconstructed Attack Chain

```text
Attacker
   |
   | Spearphishing Email
   v
Maxine Beck
   |
   | Opens Resume_WesleyTaylor.doc
   v
Microsoft Word
   |
   | Executes malicious VBA
   v
C:\ProgramData\update.js
   |
   v
wscript.exe (PID 4260)
   |
   | Downloads next stage
   v
update.exe
   |
   v
C:\Windows\Tasks\updater.exe
   |
   | PID 6216
   v
C2: 128.199.95.189:8080
   |
   v
Scheduled Task Persistence
   |
   v
Hidden PowerShell
   |
   v
Registry-Stored Encoded Payload
```

---

## Indicators of Compromise

| Type | Indicator |
|---|---|
| Email Sender | `westaylor23@outlook.com` |
| Target | `maxine.beck@quicklogisticsorg.onmicrosoft.com` |
| Attachment | `Resume_WesleyTaylor.doc` |
| MD5 | `52c4384a0b9e248b95804352ebec6c5b` |
| Malicious Script | `C:\ProgramData\update.js` |
| Malicious Executable | `C:\Windows\Tasks\updater.exe` |
| C2 | `128.199.95.189:8080` |
| Scheduled Task | `Updater` |
| Registry Path | `HKCU:\Software\Microsoft\Windows\CurrentVersion\debug` |

### Malicious URLs

```text
https://files.boogeymanisback[.]lol/aa2a9c53cbb80416d3b47d85538d9971/update.png
https://files.boogeymanisback[.]lol/aa2a9c53cbb80416d3b47d85538d9971/update.exe
```

---

## MITRE ATT&CK Mapping

| Technique | ID | Evidence |
|---|---|---|
| Spearphishing Attachment | T1566.001 | Malicious resume delivered by email |
| User Execution: Malicious File | T1204.002 | Victim opened the Word document |
| Visual Basic | T1059.005 | VBA macro execution |
| JavaScript/JScript | T1059.007 | `update.js` executed by `wscript.exe` |
| PowerShell | T1059.001 | PowerShell used in persistence chain |
| Scheduled Task | T1053.005 | `Updater` scheduled task |
| Modify Registry | T1112 | Registry used to store payload data |
| Obfuscated/Compressed Information | T1027 | Base64-encoded content |
| Application Layer Protocol | T1071 | External C2 communication |

---

## Detection Opportunities

A major behavioral indicator was the process relationship:

```text
WINWORD.EXE -> wscript.exe
```

Office applications spawning script interpreters should receive additional scrutiny, especially when followed by network activity or execution from user-writable locations.

Other useful detection and hunting opportunities include:

- Execution of scripts from `C:\ProgramData`
- Network connections from unusual executables under `C:\Windows\Tasks`
- `schtasks /Create` activity
- Scheduled tasks launching hidden PowerShell
- PowerShell decoding Base64 and invoking the result with `IEX`
- Large encoded values stored in unusual registry locations
- Connections to `128.199.95.189:8080`

---

## Key Takeaways

Boogeyman 2 required correlating several evidence sources rather than relying on a single alert or artifact.

The phishing email established **how the attacker entered the environment**. Macro analysis established **how initial code execution occurred**. The process tree established **which processes were responsible for execution**. Network artifacts established **where the malware communicated**. File scanning identified **where malicious artifacts existed**, while process-memory analysis helped reveal **how persistence was achieved**.

The biggest lesson was that Volatility plugins become significantly more useful when their results are treated as pieces of the same incident:

- `windows.pstree` — process relationships
- `windows.netscan` — network connections
- `windows.dlllist` — executable/process investigation
- `windows.filescan` — file artifacts in memory
- `windows.memmap` — process-memory extraction
- `strings` + `grep` — attacker commands, URLs, and infrastructure

Rather than treating each challenge question as an isolated answer, I used the artifacts to reconstruct a coherent intrusion:

> **Phishing → Execution → Staged Payload Delivery → C2 → Persistence**

---

## Skills Demonstrated

- Phishing email analysis
- Malicious Office document analysis
- VBA macro analysis
- Windows memory forensics
- Volatility 3
- Process-tree analysis
- Network connection analysis
- Process-memory extraction
- IOC identification and correlation
- PowerShell analysis
- Scheduled-task persistence analysis
- Registry-based persistence analysis
- Attack-chain reconstruction
- MITRE ATT&CK mapping
- SOC investigation methodology

---

## References

- [TryHackMe — Boogeyman 2](https://tryhackme.com/room/boogeyman2)
- [Volatility 3](https://github.com/volatilityfoundation/volatility3)
- [oletools](https://github.com/decalage2/oletools)
- [MITRE ATT&CK](https://attack.mitre.org/)
- [Boogeyman 2 Walkthrough — HuglerTomgaw](https://medium.com/@huglertomgaw/tryhackme-boogeyman-2-27754bfa69f6) — referenced while learning which Volatility plugins and memory-forensics approaches could be applied to the supplied memory image.

---

## Disclaimer

This investigation was performed in the authorized **TryHackMe Boogeyman 2** training environment. All malicious artifacts, infrastructure, and techniques discussed here were analyzed for educational and defensive-security purposes.
