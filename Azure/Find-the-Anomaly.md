# Find the Anomaly — Investigating a Suspicious Azure Sign-In

## Scenario

**Ticket SOC-1058:** “An account is misbehaving, somewhere in this workspace.” The night-shift handoff provided almost no context, so my task was to locate the case note, hunt down the account, and determine whether our existing monitoring would have caught the behavior without a manual investigation.

## Environment

- **Platform:** Microsoft Azure, Mad Hat Labs (live multi-user training tenant)
- **Scope:** `rg-madhat-lab-chap7` and Log Analytics workspace `law-madhat-lab-chap7`
- **Services:** Azure Monitor, Log Analytics, Azure Activity Log, Storage diagnostic settings, Azure Monitor Alerts, Workbooks
- **Tools:** Azure Portal and Kusto Query Language (KQL)
- **Access:** Visibility into the provided training resources, logs, and alert settings; no remediation changes performed
- **Dataset:** Fixed July 2026 lab events; the workbook intentionally time-smears injected events for its visualization

## Investigation

### 1. THE TICKET — Orient in an unfamiliar workspace

I started in the Log Analytics workspace without assuming which table contained the incident. I set the time range to include July 19, 2026 and ran a global search:

```kql
search "SOC-1058"
```

The query surfaced a night-shift handoff in `LabShiftHandoff_CL`, a custom table I would not have known to check first. I confirmed the ticket identifier and handoff analyst in the returned record.

**Conclusion:** Global search was the right starting point because it let me locate the case context before choosing a specific log source. I withheld the lab-specific case note from the public write-up.

![SOC handoff located by a workspace-wide KQL search](Screenshots/Find-the-Anomaly/01-ticket-handoff.png)

### 2. THE HUNT — Rank authentication failures

Next I moved to `LabSigninLogs_CL`, the custom sign-in table, and grouped failed attempts by user instead of reading each row individually:

```kql
LabSigninLogs_CL
| where TimeGenerated > ago(730d)
| where ResultType_d != 0
| summarize FailedSignins = count() by UserPrincipalName_s
| sort by FailedSignins desc
| take 10
```

The hunt identified one account with an unusual concentration of failed attempts. The suspicious user's exact UPN is omitted here to avoid disclosing the lab answer. Ranking the accounts converted a large set of similar-looking failures into a manageable lead; the count by itself was not proof of compromise.

**Conclusion:** Compare users against one another first, then investigate the outlier's full authentication sequence.

### 3. THE SEQUENCE — Check whether failures led to success

I filtered the sign-in table to the identified user and sorted by timestamp, adding `UserAgent_s` to inspect the client:

```kql
LabSigninLogs_CL
| where TimeGenerated > ago(730d)
| where UserPrincipalName_s == "<suspicious-upn>"
| project TimeGenerated, ResultType_d, IPAddress_s, AppDisplayName_s, UserAgent_s
| sort by TimeGenerated asc
```

The results showed repeated `50126` failures (invalid username or password), followed by a `0` result indicating successful authentication to Azure Portal. The failed events came from the same source IP and carried a `python-requests/2.31.0` user-agent string, consistent with scripted attempts. I inspected the successful row's user agent to complete the lab objective but have redacted its challenge-specific contents.

**Conclusion:** This is consistent with password guessing followed by successful authentication. The records justify escalation and investigation, but cannot alone establish that the account was taken over.

![Chronological sign-in activity with failures and subsequent success](Screenshots/Find-the-Anomaly/02-signin-sequence.png)

### 4. THE FIRST GAP — Storage diagnostic logging removed

To determine what the account might have accessed after authentication, I checked whether useful storage audit data even existed. The storage account `stmadhatlabchap7` had no diagnostic setting configured at the inspected scope. I then searched the management-plane activity logs for deletion of its diagnostic setting:

```kql
AzureActivity
| where TimeGenerated > ago(730d)
| where OperationNameValue endswith "DIAGNOSTICSETTINGS/DELETE"
| where _ResourceId has "stmadhatlabchap7"
    or ResourceId has "stmadhatlabchap7"
| project TimeGenerated, Caller, OperationNameValue,
          ActivityStatusValue, _ResourceId, ResourceId
| sort by TimeGenerated desc
```

The `AzureActivity` record showed a **successful deletion** on July 19, 2026, at approximately 17:40 UTC. The diagnostic setting was named `mandated storage audit`. The caller was a separate lab identity from the suspicious sign-in account, so I did **not** attribute the deletion to that account.

**Conclusion:** The deletion was proven, but its intent was not. Removing diagnostic collection is a monitoring blind spot and can be a defense-evasion technique. Without relevant storage data-plane logging, investigators may not be able to reconstruct who read the data and how during the gap. Restoring diagnostic settings later does **not** retroactively ingest events that were never collected.

![Azure Activity Log evidence of successful diagnostic-setting deletion](Screenshots/Find-the-Anomaly/03-diagnostic-deletion.png)

### 5. THE SECOND GAP — A detection rule too high to trigger

I opened `alert-failed-signins` and read the KQL condition rather than assuming a rule with the right name would catch the pattern. Its query counted failed attempts by user, then filtered with:

```kql
| where FailedCount >= 500
```

The rule required at least **500 failed attempts** per user in its evaluation window. The suspicious burst found during the hunt was below this threshold, so the rule did not surface it. I also noticed that the visible grouping was by user, not explicitly by one-hour bins; the actual evaluation period is governed by the alert configuration, not that line of KQL.

**Conclusion:** Logs can be present and searchable while alert logic still misses a meaningful anomaly. The missed detection was a tuning problem, not evidence that sign-in collection failed.

![Log search alert condition using a 500-failure threshold](Screenshots/Find-the-Anomaly/04-alert-threshold.png)

### 6. THE VISUAL CHECK — Confirm the gap using a workbook

Finally I opened the **Sign-Ins Anomaly** workbook. Its summary displayed **1,974 total events**, including **759 failed events** and **1,215 successful events**. The time chart made the abnormal increase in failed authentication traffic visible against other users' activity, despite the missed alert.

The workbook states that its KQL intentionally distributes injected lab events across a simulated 24-hour baseline. I therefore treated the chart as a visualization of the anomaly, not as an exact reconstruction of original event timing.

**Conclusion:** The workbook gave an analyst a second detection path: manually recognizing a spike missed by threshold-based alerting. The lab generated a final sign-off value; it is redacted in the screenshot and excluded from this report.

![Workbook visualizing the failed sign-in spike and event totals](Screenshots/Find-the-Anomaly/05-anomaly-workbook.png)

## What broke / what surprised me

The first obstacle was not knowing where to look. Searching only the familiar `SigninLogs` table would have missed the handoff because this lab used a custom log table. A workspace-wide search found the case note and pointed the investigation in the right direction.

The more consequential surprise was that there were **two different visibility failures**. The storage setting had been deleted, removing an expected source of telemetry; separately, sign-in telemetry existed, but the alert threshold was so high that the abnormal burst did not qualify. I could still find the sign-in pattern manually, but I could not treat missing storage logs as proof that no storage access occurred. Nor could I prove that the suspicious account deleted the diagnostic setting: the management event showed a different caller.

## Findings and recommendations

**Assessment:** One user showed an authentication pattern consistent with scripted password guessing followed by a successful login. Investigating post-login data access was complicated by absent storage diagnostic collection. The existing failed-sign-in alert was set at 500, above the observed burst. These findings warrant investigation and detection improvements; they do not by themselves prove exfiltration or establish the actor's identity.

1. **Restore diagnostic logging:** Re-enable the appropriate storage-service diagnostic categories and route them to the approved Log Analytics destination. Missing historical events cannot be recovered simply by turning the setting back on.
2. **Enforce configuration with Azure Policy:** Use a deploy-if-not-exists policy for required storage diagnostic settings and track missing configurations as non-compliant, rather than leaving resources silently unmonitored.
3. **Alert on logging being disabled:** Create an Activity Log alert for diagnostic-setting deletion operations, route it to an action group, and investigate each occurrence promptly.
4. **Retune and verify sign-in detection:** Use narrower per-account and per-IP time windows, validate thresholds against known baseline traffic, and retain a workbook view for human review when automated rules miss a spike.

## What I learned

- A global KQL `search` can locate handoff records in unfamiliar custom tables before I know the workspace schema.
- `summarize`, `sort`, and chronological `project` queries turn authentication logs into a defensible sequence of events.
- Data collection and alerting are distinct controls: either can fail independently.
- Next time, I would capture the ranked-account query and the empty diagnostic-settings blade as separate screenshots and verify the alert's evaluation period alongside its KQL threshold.

**Sign-off:** Oriented in the workspace, hunted the outlier, verified the logging removal, identified the alerting gap, and confirmed the anomaly in a workbook.

---

*This investigation was completed in an authorized multi-user Azure training tenant. Account identifiers, tenant/subscription IDs, source IP values, and challenge flags are intentionally omitted or redacted from the public evidence.*
