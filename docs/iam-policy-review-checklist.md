# IAM Policy Review Checklist

A practical checklist for reviewing AWS IAM configurations in any environment. Derived from hands-on misconfiguration detection and analysis work. Use this during IAM audits, access reviews, or as part of a cloud security baseline assessment.

---

## 1. Policy Design

- [ ] No wildcard `Action: *` permissions exist in any custom policy
- [ ] No wildcard `Resource: *` permissions exist without explicit justification
- [ ] All custom policies scope both `Action` and `Resource` as narrowly as possible
- [ ] No inline policies attached to users, groups, or roles (prefer managed policies for auditability)
- [ ] Policies follow the principle of least privilege — each identity has only the permissions required for its function
- [ ] AWS managed policies are used where appropriate rather than reinventing equivalent custom policies
- [ ] Policies have been reviewed for permission creep (access granted historically that is no longer needed)

---

## 2. User & Group Management

- [ ] No policies attached directly to IAM users — all permissions managed via groups or roles
- [ ] Users are organised into groups that reflect their job function or access tier
- [ ] Each group has a clearly defined and documented purpose
- [ ] No user belongs to more groups than necessary
- [ ] Unused IAM users (no console login or API activity in 90+ days) are flagged for removal or deactivation
- [ ] Root account is not used for day-to-day operations
- [ ] Root account has MFA enabled and access keys are deleted or not created
- [ ] IAM admin user exists for account management tasks instead of using root

---

## 3. Access Control Architecture

- [ ] IAM roles are used for service-to-service access instead of long-lived IAM user credentials
- [ ] EC2 instances, Lambda functions, and other compute resources use instance profiles / execution roles, not embedded access keys
- [ ] Cross-account access uses roles with explicit trust policies, not shared credentials
- [ ] Privilege escalation paths have been reviewed — no user can attach a more permissive policy to themselves
- [ ] No user has both `iam:CreatePolicy` and `iam:AttachUserPolicy` without additional controls

---

## 4. Credential Hygiene

- [ ] MFA is enforced for all human IAM users, especially those with console access
- [ ] Access keys are rotated at least every 90 days
- [ ] No access keys exist for the root account
- [ ] Unused access keys (inactive 30+ days) are disabled or deleted
- [ ] Credentials are never hardcoded in source code, scripts, or config files
- [ ] Access key usage is monitored via CloudTrail for anomalous patterns

---

## 5. Logging & Monitoring

- [ ] AWS CloudTrail is enabled across all regions (multi-region trail)
- [ ] Global service events are enabled to capture IAM activity (IAM is a global service)
- [ ] CloudTrail logs are stored in an S3 bucket with appropriate retention (beyond the default 90-day Event History limit)
- [ ] S3 log bucket has encryption enabled (at minimum SSE-S3)
- [ ] S3 log bucket has access logging enabled and is not publicly accessible
- [ ] CloudTrail log integrity validation is enabled
- [ ] Management events (Read and Write) are captured — not just Write events
- [ ] Alerts or notifications are configured for key events (e.g. root login, policy changes, new user creation)

---

## 6. Detection & Response Indicators

Watch for the following API call patterns in CloudTrail logs — these are common indicators of reconnaissance or privilege escalation activity:

| API Call Pattern | What It May Indicate |
|---|---|
| High volume of `Describe*` calls | Resource enumeration across services |
| High volume of `List*` calls | Service and bucket/resource discovery |
| `GetBucketAcl`, `GetBucketPolicy` | S3 access probing |
| `ListAttachedUserPolicies` | Checking own or others' permissions |
| `CreatePolicy`, `AttachUserPolicy` in sequence | Potential privilege escalation attempt |
| Cross-service API calls from a single identity | Lateral movement indicators |
| API activity from unfamiliar IP or region | Possible credential compromise |
| Console logins without MFA | Policy violation or misconfiguration |

---

## 7. Periodic Review Tasks

These tasks should be performed on a defined schedule as part of ongoing IAM governance:

**Monthly**
- [ ] Review IAM Access Analyzer findings for overly permissive policies
- [ ] Check for unused credentials (AWS Credential Report)
- [ ] Review any new custom policies created in the period

**Quarterly**
- [ ] Full access review — validate all users still require their current access
- [ ] Review group memberships and remove users who have changed roles
- [ ] Rotate access keys that are approaching 90 days

**Annually**
- [ ] Full IAM policy audit — review all custom policies against current least-privilege requirements
- [ ] Re-evaluate AWS managed policies in use — check for updates or better-scoped alternatives
- [ ] Review and update IAM documentation and ownership assignments

---

## 8. Compliance Alignment

This checklist maps to the following control frameworks:

| Control Area | NIST CSF | CIS AWS Benchmark | ISO 27001 |
|---|---|---|---|
| Least privilege | PR.AC-4 | 1.1 – 1.16 | A.9.2 |
| MFA enforcement | PR.AC-7 | 1.5, 1.6 | A.9.4.2 |
| Credential management | PR.AC-1 | 1.12 – 1.14 | A.9.2.6 |
| Logging & monitoring | DE.CM-3 | 3.1 – 3.5 | A.12.4 |
| Access review | PR.AC-6 | 1.1 | A.9.2.5 |

---

## References

- [AWS IAM Best Practices — AWS Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [CIS Amazon Web Services Foundations Benchmark](https://www.cisecurity.org/benchmark/amazon_web_services)
- [NIST SP 800-53 — Access Control Family](https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final)
- [AWS IAM Access Analyzer](https://docs.aws.amazon.com/IAM/latest/UserGuide/what-is-access-analyzer.html)

---

*Derived from the AWS IAM Misconfiguration Detection & Analysis lab. For project context, see the main [README](../README.md).*
