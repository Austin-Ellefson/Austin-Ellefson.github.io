# The Privilege Audit

### Auditing Privileged Access Across Azure

## Scenario

This assessment focused on auditing privileged access across a live Azure environment.

Rather than investigating a known security incident, the goal was to determine **who had privileged access, where that access applied, and whether the assignments were necessary and properly managed**.

I performed the audit using multiple Azure methods to compare what each could reveal and where each had visibility gaps.

---

## Environment

* **Platform:** Microsoft Azure / Microsoft Entra ID
* **Environment:** Live multi-user Azure training tenant
* **Access:** Reader access, plus one PIM-eligible role
* **Services reviewed:** Azure RBAC, Microsoft Entra ID, Privileged Identity Management (PIM)
* **Tools used:**

  * Access Control (IAM)
  * Azure CLI
  * Azure Resource Graph Explorer / KQL
  * Microsoft Entra Privileged Identity Management (PIM)
* **Audit focus:** Privileged role assignments, excessive access, orphaned assignments, standing vs. eligible privilege, and assignment scope

> **Note:** Sensitive lab values have been intentionally redacted. Challenge flags, RoleAssignmentDescription values, orphaned principal IDs, and other challenge-specific identifiers are not included.

---

# Scope and Methodology

Azure provides several methods for reviewing role assignments, but each method exposes different information.

For this audit, I reviewed privileged access using four Azure tools and then performed a final manual hunt using the information gathered throughout the assessment.

My goal was not only to identify questionable access but also to understand **which audit method was best suited for each task and what each method could miss.**

## Methodology Comparison

| Method                   | What It Shows                                                                           | What It Can Miss                                                                  |
| ------------------------ | --------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| **IAM blade / export**   | Active assignments at a selected scope, including inherited assignments                 | Group membership and easily overlooked orphaned principals                        |
| **Azure CLI**            | Scriptable RBAC assignment data, including fields that can expose unresolved principals | Typically requires querying individual scopes unless additional scripting is used |
| **Resource Graph (KQL)** | Role assignments across the Azure environment in a single query                         | Eligible PIM assignments                                                          |
| **PIM export**           | Eligible vs. active privileged assignments and activation history                       | Standing assignments that were never managed through PIM                          |

---

# Audit

## 1. Establishing an Access Baseline with IAM

### Method: Access Control (IAM) Blade

I started the privilege audit with Azure's **Access Control (IAM)** blade.

Every Azure scope has its own IAM view, including management groups, subscriptions, resource groups, and individual resources. This makes IAM the simplest starting point for determining which identities currently have access and what roles they hold.

Rather than manually inspecting every assignment, I reviewed the **Download role assignments** export. Exporting role assignments from the subscription level provides a broader view because the report can include inherited assignments as well as assignments associated with child resources.

Because my operative account had restricted visibility, I could not generate the complete export with identity names myself. For this portion of the audit, I reviewed a provided role-assignment CSV representing the data an auditor would normally obtain from the subscription-level export.

### What I Looked For

I reviewed the role-assignment export for:

* Highly privileged roles such as **Owner**
* Users with privileged access across multiple scopes
* Direct user assignments versus group-based access
* Assignments inherited from higher scopes
* Repeated or potentially redundant assignments
* Unusual assignments or identities requiring further investigation

### What I Found

The export established my initial baseline of active privilege within the subscription.

During the review, I identified a user account holding the **Owner** role at the subscription level. I also observed additional Owner assignments associated with the same identity at lower scopes in the environment.

A subscription-level Owner assignment stood out because permissions granted at a higher scope can apply to resources beneath it. This raised the question of whether the additional lower-scope assignments were necessary or whether the account had been granted more access than required.

At this stage, I did **not** assume that the assignments were malicious or incorrectly configured. Instead, I treated the pattern as something that warranted further investigation using the other audit methods.

### Blind Spot

The IAM blade and role-assignment export provide a useful view of **active RBAC assignments**, but they do not provide the complete privilege picture.

For example, an assignment may show that a group has access without showing which individual users receive those permissions through group membership.

Large exports can also make stale or unresolved identities difficult to recognize manually. An abnormal assignment can easily be buried among legitimate role assignments.

This meant I needed additional tooling to continue the audit.

### Conclusion

The IAM blade and role-assignment export were effective for establishing an initial access baseline and identifying accounts that deserved deeper investigation.

However, the method was better suited for answering:

> **"Who currently has access at this scope?"**

than determining whether every assignment was still valid, necessary, or properly managed.

The privilege pattern identified here gave me a starting point for the next phase of the audit, where I used **Azure CLI** to inspect role-assignment data more directly.

### Evidence

#### Role Assignment Export

![Azure IAM Role Assignment Export](images/iam-role-assignment-export.png)

*The role-assignment export provided a subscription-level view of active Azure RBAC assignments. Sensitive lab values, challenge flags, assignment descriptions, and identifiers have been redacted.*

---

## 2. Auditing Role Assignments with Azure CLI

`[TO BE COMPLETED]`

---

## 3. Expanding the Audit with Azure Resource Graph

`[TO BE COMPLETED]`

---

## 4. Reviewing Eligible Privilege with PIM

`[TO BE COMPLETED]`

---

## 5. The Hunt

`[TO BE COMPLETED]`

---

# What Broke / What Surprised Me

`[ADD YOUR REAL DEAD ENDS, WRONG ASSUMPTIONS, TOOLING ISSUES, OR SURPRISES HERE.]`

Some questions to consider while completing the lab:

* Did one tool show something another did not?
* Did you initially investigate the wrong scope?
* Was something difficult to identify in the portal but obvious through CLI?
* Did Resource Graph return something different from what you expected?
* Did PIM change your understanding of who actually had privileged access?
* What took you the longest to figure out?

---

# Findings and Recommendations

## Finding 1 — `[FINDING]`

**Severity:** `[SEVERITY]`

`[DESCRIBE THE FINDING.]`

**Recommendation:** `[RECOMMENDATION]`

---

## Finding 2 — `[FINDING]`

**Severity:** `[SEVERITY]`

`[DESCRIBE THE FINDING.]`

**Recommendation:** `[RECOMMENDATION]`

---

## Finding 3 — `[FINDING]`

**Severity:** `[SEVERITY]`

`[DESCRIBE THE FINDING.]`

**Recommendation:** `[RECOMMENDATION]`

---

# What I Learned

* `[TECHNICAL LESSON]`
* `[TOOLING LESSON]`
* `[IAM / GOVERNANCE LESSON]`
* `[AUDITING LESSON]`
* `[WHAT I WOULD DO DIFFERENTLY NEXT TIME]`
