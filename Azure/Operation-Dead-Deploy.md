Investigated a mislabeled resource group in azure.

A junior intern was given temporary Contributor access to deploy a test environment in the Mad Hat Labs Azure subscription but created the resources without following the organization’s governance standards.
As the on-call Azure engineer with Reader access, I investigated the environment to determine what was deployed, identify which governance controls failed or were bypassed, and document the resulting risks and evidence.

-  Microsoft Azure
-  Live multi-user Azure training tenant
-  Reader access
-  Azure Resource Groups, Virtual Machines, Storage Accounts, Virtual Networks, and Network Security Groups
-  Azure Policy, resource tagging, naming standards, security configuration, and resource organization
-  Azure Portal, Activity Log, and Azure Policy

## Investigation

1. I opened the portal and began looking at resource groups to see if any looked out of place. Towards the bottom I found my culprit.
   ![Resource Groups](Screenshots/S1.png)
3. After this I opened the RG and saw that it had only one resource deployed. I opened that and inspected the tags. I found that its owner was the intern.
  ![tags](Screenshots/S2.png)
3. I wanted to determine how the intern's resource was originally created, so I returned to the resource group and reviewed its deployment history.
   Since Azure Resource Manager records deployments made against a resource group, I used the Deployments blade to trace the resource back to the deployment that created it.
   I reviewed the deployment details, including its name, timestamp, parameters, and deployment status.
   The deployment name provided additional evidence connecting the resource to the intern and helped establish when the environment was created.
   ![Deployments](Screenshots/S3.png)
4. After confirming how the resource was deployed, I investigated why Azure Policy did not prevent the improperly named resource from being created.
   I found that the naming convention policy was active and correctly identified the resource as non-compliant, but it had still allowed the deployment to succeed. I reviewed the policy assignment and found that its **Effect** was set to `Audit` instead of `Deny`.
 This explained the governance failure: the policy was configured to detect and report naming violations rather than block them, allowing the intern's non-compliant resource to be created.
   ![Naming Policy Effect Set to Audit](Screenshots/S4.png)

## What broke / what surprised me
The most credible section in the document. Dead ends, wrong guesses, the thing that took an hour. Employers know real work is messy. This section separates you from certificate collectors.

## Findings and recommendations
What you determined, plus 2 or 3 recommendations as if you were reporting to the resource owner.

## What I learned
3 to 5 bullets. At least one technical, one "what I'd do differently."
