# Spin Up and Lock Down — Reviewing an Azure Deployment Before Production

## Scenario

A newly deployed Function App in the Mad Hat Labs training environment needed a security review before promotion to production. I reviewed its configuration against an existing production baseline to identify potential misconfigurations, verify exposure where possible, and recommend which issues should be addressed first.

## Environment

- **Platform:** Microsoft Azure
- **Environment:** Live multi-user Azure training tenant (Mad Hat Labs)
- **Access level:** Reader
- **Services reviewed:** Azure Functions, Azure Storage, managed identities, and network access controls (as investigated)
- **Tools:** Azure Portal; browser private session for any permitted anonymous-access validation
- **Review method:** Compare the new deployment with an existing working production stack; no written configuration baseline was provided.

## Investigation

### 1. THE WHERE — Is the service deployed in the right region?

I started by opening the new Function App's **Overview** page in the Azure Portal. I noticed that the app was deployed in **Australia East**. This stood out because I expected it to be near the other services in the stack.

A Function App running far from its dependencies could introduce network latency and potentially cross-region data-transfer charges. However, the location alone does not prove a mismatch, so my next step is to check the regions of the existing production Function App and the storage or data services it uses.

**Status:** Australia East deployment confirmed; production-baseline comparison pending.

**Evidence — Function App overview showing Australia East:**

![Redacted Azure Function App overview showing deployment in Australia East](Screenshots/Spin-Up-S1-redacted.png)

### 2. THE WHO — How does the application authenticate to its dependencies?

I opened the **Identity** blade on the newly deployed Function App and checked both **System assigned** and **User assigned**. Neither section showed an enabled or attached managed identity.

To establish what the working environment looked like, I checked the **User assigned** tab on the production Function App. Unlike the new deployment, the production app had a user-assigned managed identity attached. This gave me a concrete configuration difference rather than relying on a generic assumption about how the application should be set up.

**Why it matters:** Managed identities let Azure resources obtain access tokens for supported services without storing application secrets. The new app's missing identity raises a question about whether it uses connection strings, account keys, or another authentication method to reach storage. The identity comparison alone does **not** prove that credentials are stored in the app; that would require checking its configuration and dependencies.

**Recommendation:** Confirm how the new Function App connects to storage. If it relies on a stored key or connection string, work with the application owner to configure an appropriately scoped managed identity and use identity-based access, then remove credentials that are no longer necessary.

**Status:** Difference from production baseline confirmed; storage authentication method not yet verified.

**Evidence:** Production Function App Identity blade showing a user-assigned identity (redacted screenshot to be uploaded as `Screenshots/Spin-Up-S2-Baseline-Redacted.png`).

### 3. THE LEAK — Is any storage data anonymously accessible?

_Pending investigation: Inspect storage account and container public-access settings. If permitted, verify any suspected public blob access in a private browser window without authenticating. Do not publish sensitive blob URLs._

### 4. THE DOOR — Who can reach the application?

_Pending investigation: Review Function App inbound access restrictions, default actions, and how they compare with the existing production configuration._

### 5. THE CALL — What should be fixed first?

_Pending completed review. Rank findings based on demonstrated exposure, impact, and likelihood rather than treating every configuration difference as equally urgent._

**Prioritization principle:** Exposure beats hygiene; active exposure beats potential risk.

## What broke / what surprised me

_To be completed with actual dead ends, incorrect assumptions, and troubleshooting during this review._

## Findings and recommendations

**Preliminary observation:** The new Function App is running in Australia East. Determine whether this differs from the production baseline and the location of its dependencies before categorizing it as a deployment defect.

_Final recommendations will follow the evidence from the identity, storage exposure, and inbound-access reviews. Consider documenting a repeatable pre-deployment checklist to prevent the same problems in future releases._

## What I learned

_To be completed after investigation with 3–5 takeaways, including one technical lesson and one thing I would do differently._

---

> **Publication note:** Redact subscription IDs, tenant IDs, sensitive resource identifiers, challenge flags, access tokens, and sensitive URLs from screenshots or prose before publishing. Evidence images must be uploaded separately to the linked Screenshots path.
