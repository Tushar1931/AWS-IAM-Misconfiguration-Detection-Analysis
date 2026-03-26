#  AWS IAM MISCONFIGURATION DETECTION & ANALYSIS

> A hands-on AWS security lab simulating a real-world IAM privilege escalation attack — from intentional misconfiguration through suspicious activity simulation, CloudTrail log analysis, and full remediation. Built to demonstrate identity security fundamentals and cloud threat detection skills.

## Table of Contents
 
- [Project Overview](#-project-overview)
- [Objectives](#-objectives)
- [Architecture](#-architecture)
- [Tools & Technologies](#-tools--technologies)
- [Step-by-Step Setup](#-step-by-step-setup)
  - [Step 1 — IAM User & Group Configuration](#step-1--iam-user--group-configuration)
  - [Step 2 — Intentional Misconfiguration](#step-2--intentional-misconfiguration)
  - [Step 3 — Enable CloudTrail Logging](#step-3--enable-cloudtrail-logging)
  - [Step 4 — Simulate Suspicious Activity](#step-4--simulate-suspicious-activity)
- [Log Analysis & Findings](#-log-analysis--findings)
- [Security Interpretation](#-security-interpretation)
- [Key Finding](#-key-finding)
- [Remediation](#-remediation)
- [Lessons Learned](#-lessons-learned)


## Project Overview
 
This project simulates a realistic **IAM privilege escalation scenario** in AWS, where a user with restricted permissions is granted an over-permissive policy, bypassing least privilege controls. The lab demonstrates how attackers exploit misconfigured IAM policies to enumerate resources and escalate access across AWS services.
 
The full attack lifecycle is captured and analysed using **AWS CloudTrail**, showcasing hands-on skills in identity security, cloud threat detection, and log forensics.
 
This project reflects real-world scenarios encountered in **Identity Security Analyst**, **Cloud Security Engineer**, and **GRC/Compliance** roles — where IAM misconfigurations remain one of the leading causes of cloud data breaches.
 
---
 
## Objectives
 
- Design a least-privilege IAM architecture using users, groups, and policies
- Intentionally introduce a critical misconfiguration (wildcard `*:*` permissions) to simulate poor IAM hygiene
- Enable persistent, multi-region audit logging via AWS CloudTrail
- Simulate reconnaissance and privilege escalation behaviour from a misconfigured account
- Analyse CloudTrail logs to detect and interpret suspicious API activity
- Apply remediation and document security recommendations
 
---

## Architecture
 
### User Role Design
 
| User | Group | Policy | Purpose |
|---------|---|---|---|
| `analyst-user` | `LimitedAccess` | `ViewOnlyAccess` (AWS Managed) | Baseline — legitimate least-privilege user |
| `dev-user` | `LimitedAccess` | `ViewOnlyAccess` + **Custom `*:*` policy (direct attach)** | Simulates misconfigured / compromised account |
 
### Why This Design Matters
 
The `analyst-user` represents **correct IAM design** — permissions granted via group membership with a scoped managed policy. The `dev-user` demonstrates a **common real-world failure**: a wildcard custom policy attached directly to a user, overriding the group level least-privilege design. This is a textbook example of how privilege escalation occurs without an explicit attacker action, purely through misconfiguration.
 
---

## Tools & Technologies
 
| Tool | Purpose |
|---|---|
| **AWS IAM** | User, group, and policy management |
| **AWS CloudTrail** | API-level audit logging and event history |
| **AWS S3** | Persistent log storage |
| **AWS EC2** | Accessed during simulation to verify unrestricted service access |
| **AWS Console** | Configuration, simulation, and log analysis |
 
---

## Step-by-Step Setup
 
### Step 1 — IAM User & Group Configuration
 
**Created an IAM admin user** (to avoid using root account):
 
```
IAM → Users → Create user → Attach AdministratorAccess
Enable console access → save credentials
```
 
**Created two normal IAM users for simulation:**
 
**`analyst-user`** — represents a legitimate, least-privilege account:
- Assigned to `LimitedAccess` group
- Group policy: `ViewOnlyAccess` (AWS managed)
 
**`dev-user`** — used to simulate suspicious/misconfigured activity:
- Assigned to `LimitedAccess` group (same ViewOnlyAccess baseline)
- Will have an over-permissive policy directly attached in Step 2
 
**Group setup:**
```
Group: LimitedAccess
  └── Policy: ViewOnlyAccess (AWS Managed)
       └── Members: analyst-user, dev-user
```
 
> 💡 **Design rationale:** Starting both users in the same group establishes a clear baseline. The impact of the misconfiguration becomes measurable, and the security comparison between the two users becomes clear during log analysis.
 
---
 
### Step 2 — Intentional Misconfiguration
 
**Created a custom over-permissive IAM policy:**
 
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "*",
      "Resource": "*"
    }
  ]
}
```
 
**Attached this policy directly to `dev-user`** — bypassing the group-based least-privilege design.
 
 **Why this is a critical misconfiguration:**
- Violates the **principle of least privilege**
- Grants unrestricted access across all AWS services and resources
- Represents **direct policy attachment overriding group-level access controls**
- This pattern is a leading root cause of cloud security breaches and privilege escalation attacks
  
---
 
### Step 3 — Enable CloudTrail Logging
 
Created a CloudTrail trail named **`SecurityTrail`** with the following configuration:
 
| Setting | Value |
|---|---|
| Trail type | **Multi-region trail** |
| Event types | Management events (Read + Write) |
| Log destination | Auto-created S3 bucket |
| Encryption | SSE-S3 (default) |
| IAM events | Captured (global service events enabled) |
 
**Why multi-region?**
IAM is a global AWS service. To capture IAM API activity, the trail must be configured to log **global service events**. A single-region trail would miss IAM actions entirely.
 
**What is logged vs. what is not:**
 
| Event Type | Logged? |
|---|---|
| Management events (API calls, console activity) | ✅ Yes |
| IAM activity (global service events) | ✅ Yes |
| Data events (S3 object-level, Lambda invocations) | ❌ Not enabled |
| Insights events | ❌ Not enabled |
 
> **Note:** AWS retains only 90 days of management events via Event History. The CloudTrail trail was configured to enable **persistent logging beyond 90 days** and allow deeper forensic analysis from S3.
 
**Log verification:**
- Accessed Event History in the CloudTrail console
- Confirmed logs were being generated immediately
- Observed initial API calls including `DescribeInstances`, `DescribeRegions`, `ListIndexes`, `GetBucketAcl`
 
> **Tip:** If the trail doesn't show logs immediately, verify the S3 bucket policy allows CloudTrail to write, and confirm global service events are enabled under trail settings.
 
---
 
### Step 4 — Simulate Suspicious Activity
 
Logged in as `dev-user` and performed the following actions to simulate attacker reconnaissance behaviour:
 
**IAM Enumeration:**
- Listed IAM users, policies, and roles
- Performed actions consistent with an attacker mapping their access scope
 
**Privilege Escalation Verification:**
- Confirmed that the `*:*` custom policy granted unrestricted access across services
 
**Service Access (Lateral Movement Simulation):**
- Accessed **S3** — listed buckets and ACLs
- Accessed **EC2** — described instances, regions, load balancers, and alarms
- Accessed additional AWS services confirming unrestricted cross-service access
 
---

## Log Analysis & Findings
 
After simulating the activity, CloudTrail logs were filtered by `dev-user` in the Event History console.
 
### Observed API Calls
 
```
DescribeRegions
DescribeInstances
DescribeLoadBalancers
DescribeAlarms
ListBuckets
GetBucketAcl
ListIndexes
GetBucketPolicy
```
 
### What These Calls Indicate
 
| API Call Pattern | Security Signal |
|---|---|
| `Describe*` across multiple services | **Resource enumeration** — mapping what exists |
| `List*` on S3 | **Service exploration** — identifying accessible buckets |
| `Get*` on IAM/S3 | **Permission probing** — checking what can be read |
| Cross-service activity (EC2 + S3 + IAM) | **Lateral movement indicators** |
 
>  **Important:** CloudTrail logs **API-level activity**, not UI interactions. Every console click that triggers an AWS API call is logged — meaning the full scope of `dev-user`'s actions is captured in the trail, even when using only the AWS Console.
 
---

## Security Interpretation
 
The activity observed in CloudTrail logs is consistent with **early-stage attacker reconnaissance behaviour**:
 
- The user explored IAM and AWS resources systematically
- Actions were performed **beyond typical least-privilege usage**
- Activity showed **enumeration behaviour across multiple AWS services**
- The user accessed EC2, S3, and account-level information
- The overall pattern is consistent with the **reconnaissance phase of an attack kill chain**
 
### Detection Insight
 
Suspicious behaviour was identified through:
 
- **High volume of `Describe*` and `List*` API calls** in a short timeframe — a known indicator of automated or scripted reconnaissance
- Cross-service API calls from a single user identity — unusual for a `dev-user` with no legitimate need to access EC2, S3 load balancers, and IAM simultaneously
- These patterns can indicate:
  - Reconnaissance activity by a malicious insider
  - An attacker using compromised credentials
  - Early-stage cloud environment exploration prior to data exfiltration or persistence
 
---
 
## Key Finding
 
> **`dev-user` was able to perform unrestricted actions across all AWS services due to an over-permissive IAM policy (`*:*`) attached directly to the user.**
 
This finding demonstrates three compounding failures:
 
1. **Bypassed group-based least-privilege controls** — direct policy attachment overrode the intended group design
2. **Excessive access across multiple AWS services** — no boundary on what the user could read, write, or delete
3. **No compensating controls** — without CloudTrail analysis, this misconfiguration would have gone undetected
 
**Over-permissive IAM policies are a leading cause of cloud security breaches and privilege escalation attacks.** This scenario mirrors real-world incidents where developers or service accounts are granted `*:*` permissions "temporarily" and the change is never rolled back.
 
---
 
## Remediation
 
### Actions Taken
 
 **Removed** the direct attachment of the over-permissive `*:*` custom policy from `dev-user`
 
### Recommendations
 
| Recommendation | Detail |
|---|---|
| **Enforce least privilege** | Never assign permissions broader than what the role requires |
| **Use group-based permissions only** | Avoid direct policy attachments to users — manage access via groups or roles |
| **Avoid wildcard `*:*` permissions** | In production environments, always scope `Action` and `Resource` explicitly |
| **Implement IAM Access Analyzer** | Continuously monitor for overly permissive policies across the account |
| **Conduct periodic IAM reviews** | Regularly audit attached policies, especially custom ones |
| **Enable AWS Config rules** | Use rules like `iam-no-inline-policy` and `iam-user-no-policies-check` to automate detection |
| **Use IAM roles over IAM users** | For service-to-service access, prefer roles with short-lived credentials |
 
---

## 💡 Lessons Learned
 
### Technical
 
- **IAM is a global service** — CloudTrail must be configured with global service events enabled, otherwise IAM API calls are invisible in single-region trails
- **Direct policy attachment is dangerous** — it silently overrides group-level controls and is easy to miss in audits
- **CloudTrail logs API calls, not UI actions** — every console interaction that triggers an API is logged, but purely client-side UI browsing is not. This distinction matters for forensics
- **`Describe*` and `List*` API call patterns are reliable reconnaissance indicators** — security tools like AWS GuardDuty and SIEM platforms use these as detection signals
- **AWS retains only 90 days of event history** — production environments need persistent trails to S3 for compliance and forensic investigations
 
### Operational
 
- A misconfiguration doesn't need a malicious actor to create a breach — poor IAM hygiene alone creates the attack surface
- The gap between `analyst-user` (correct design) and `dev-user` (misconfigured) shows how quickly access controls can be undermined by a single policy attachment
- Log analysis is only useful if logging was configured correctly from the start — security must be proactive, not reactive
 
---

## Author
 
**Tushar Sharma**  
Information Security Professional | IAM & GRC Specialist
