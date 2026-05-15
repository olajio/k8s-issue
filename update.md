Let me tackle all three.

---

## 1. Testing Without Updating RolesAnywhere

Yes — you can test by assuming the role manually with `sts:AssumeRole` and injecting `abac_code` as a session tag directly. This simulates exactly what RolesAnywhere *would* do if the profile had the tag configured.

```bash
# Step 1: Assume the role manually with the abac_code session tag injected
aws sts assume-role \
  --role-arn arn:aws:iam::469620122115:role/aws_ELK \
  --role-session-name abac-test-session \
  --tags Key=abac_code,Value=620 \
  --region us-east-2
```

This returns temporary credentials. Export them:

```bash
export AWS_ACCESS_KEY_ID=<AccessKeyId from output>
export AWS_SECRET_ACCESS_KEY=<SecretAccessKey from output>
export AWS_SESSION_TOKEN=<SessionToken from output>
```

Then test the github secret:

```bash
# Test describe (abac_operator or abac_admin match)
aws secretsmanager describe-secret \
  --secret-id arn:aws:secretsmanager:us-east-2:469620122115:secret:github_hsv_internal/itsma/service_elastic_auto_HSV-SRVK6D \
  --region us-east-2

# Test get-secret-value (abac_operator match)
aws secretsmanager get-secret-value \
  --secret-id arn:aws:secretsmanager:us-east-2:469620122115:secret:github_hsv_internal/itsma/service_elastic_auto_HSV-SRVK6D \
  --region us-east-2
```

If both succeed → your ABAC analysis is correct and the RolesAnywhere profile tag is the only missing piece. Clean up after:

```bash
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
```

> **Note:** This `sts:AssumeRole` test only works if your current CLI identity has permission to assume `aws_ELK` and pass session tags. If you're running this from a personal IAM user or admin role you already have, it should work fine.

---

## 2. Message to Cloud Automation Team---

## 3. Jira Card Update

```
## Investigation Summary

**Issue:** aws_ELK role (assumed via IAM Roles Anywhere) is unable to access the secret
github_hsv_internal/itsma/service_elastic_auto_HSV-SRVK6D despite the role having
the AbacAccess policy attached.

---

## Root Cause Identified

The AbacAccess IAM policy uses Attribute-Based Access Control (ABAC). It grants access
to resources only when the resource tag (e.g. abac_operator) contains the caller's
session tag aws:PrincipalTag/abac_code.

The aws_ELK role is assumed via IAM Roles Anywhere using the aws_signing_helper
credential process on the ELK host. The current RolesAnywhere profile configuration
does not inject any session tags into the assumed role session. Specifically, the
abac_code session tag is never set.

Because aws:PrincipalTag/abac_code is always empty, the ABAC condition evaluates to
false for every resource — making the AbacAccess policy effectively inactive for this
role.

This was confirmed by:
  - aws sts get-caller-identity → role assumed correctly
  - describe-secret on onprem/logstash/automation → SUCCESS (direct ARN grant,
    no ABAC involved)
  - describe-secret on github_hsv_internal/itsma/service_elastic_auto_HSV → DENIED
    (no identity-based policy allows the action — ABAC condition not met)

---

## Why Adding Tags to the Secret Alone Will Not Fix This

The cloud automation team advised adding abac_admin and abac_operator tags to the
github secret. While the secret does need those tags, this addresses only one side
of the ABAC condition (the resource side). The session side (abac_code on the caller)
is still missing, so the condition will still evaluate to false.

Reference: onprem/logstash/automation has the same abac_operator/abac_admin tags and
this role still cannot access it via ABAC — it only works because of the separate
ElkSecretCustomAccess direct-ARN policy.

---

## Two-Part Fix Required

  Part 1 — Secret resource tags (Cloud Automation)
  Add to github_hsv_internal/itsma/service_elastic_auto_HSV:
    abac_operator = 620
    abac_admin    = 620

  Part 2 — RolesAnywhere Profile session tag (Cloud Automation / Terraform)
  Update the aws_rolesanywhere_profile Terraform resource to inject abac_code = 620
  as a session tag when aws_ELK is assumed.

  Profile IDs to update:
    us-east-2 (primary): c68bd204-2405-4398-85b2-01e65113e35d
    us-east-1 (DR):      a81ce7f6-e974-4784-9c53-d24b8ca2b4ee

---

## Validation Plan

Once Part 2 is deployed, verify on the ELK host:

  aws secretsmanager describe-secret \
    --secret-id arn:aws:secretsmanager:us-east-2:469620122115:secret:github_hsv_internal/itsma/service_elastic_auto_HSV-SRVK6D \
    --region us-east-2

  aws secretsmanager get-secret-value \
    --secret-id arn:aws:secretsmanager:us-east-2:469620122115:secret:github_hsv_internal/itsma/service_elastic_auto_HSV-SRVK6D \
    --region us-east-2

Expected result: both commands succeed without AccessDeniedException.

---

## Status
Pending Cloud Automation team to update RolesAnywhere profile via Terraform (Part 2).
Secret tags (Part 1) also pending confirmation from Cloud Automation.
```
