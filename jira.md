```
## Investigation Summary

**Issue:** aws_ELK role (assumed via IAM Roles Anywhere) is unable to access the secret
github_hsv_internal/itsma/service_elastic_auto_HSV-SRVK6D despite the role having
the AbacAccess policy attached.

---

## Testing Performed

Two secrets were tested from the ELK host (ms51-22elkalt01) using the aws_ELK role
assumed via RolesAnywhere. Both secrets have identical tags:

  abac_admin:    620 710
  abac_operator: 620 710
  Environment:   shd

Logstash secret (onprem/logstash/automation-9M8cvG)
  $ aws secretsmanager describe-secret \
      --secret-id arn:aws:secretsmanager:us-east-2:469620122115:secret:onprem/logstash/automation-9M8cvG \
      --region us-east-2
  Result: SUCCESS ✅

GitHub secret (github_hsv_internal/itsma/service_elastic_auto_HSV-SRVK6D)
  $ aws secretsmanager describe-secret \
      --secret-id arn:aws:secretsmanager:us-east-2:469620122115:secret:github_hsv_internal/itsma/service_elastic_auto_HSV-SRVK6D \
      --region us-east-2
  Result: AccessDeniedException ❌
  "User: arn:aws:sts::469620122115:assumed-role/aws_ELK/4f00001a2aaa67d0249689f47d000000001a2a
  is not authorized to perform: secretsmanager:DescribeSecret on resource:
  github_hsv_internal/itsma/service_elastic_auto_HSV-SRVK6D because no
  identity-based policy allows the action"

---

## Root Cause Analysis

Despite having identical tags, the two secrets are covered by completely different
policies — which is why one works and the other doesn't.

  Logstash secret → covered by ElkSecretCustomAccess (direct hardcoded ARN grant)
    No tag matching involved. ABAC is completely irrelevant to this secret.
    It would work regardless of what tags exist on the resource or the session.

  GitHub secret → no dedicated policy. Only coverage is AbacAccess, which uses
    an ABAC condition:

      aws:ResourceTag/abac_operator  must contain  aws:PrincipalTag/abac_code

    The right side of this condition — aws:PrincipalTag/abac_code — must be present
    as a session tag on the caller at the time the role is assumed. The aws_ELK role
    is assumed via IAM RolesAnywhere, and the current RolesAnywhere profile does not
    inject any session tags. abac_code is therefore always empty on the session, so
    the ABAC condition never evaluates to true regardless of what tags are on the
    resource.

This also explains why the cloud automation team's initial recommendation to add
abac_admin and abac_operator tags to the GitHub secret would not have resolved the
issue — those tags only affect the left side of the ABAC condition. The missing
abac_code session tag on the right side means the condition still fails.

---

## Additional Finding: Role Tag vs Session Tag Mismatch

The aws_ELK IAM role itself has the following tag:
  abac_code: 118

However, IAM role-level tags do NOT automatically flow into the assumed role session
as principal tags. Only tags explicitly injected at session creation time (via
RolesAnywhere profile attribute mapping or --session-tags) become
aws:PrincipalTag/* values that ABAC conditions can evaluate against.

Furthermore, even if role tags did flow into the session, the value 118 does not
appear in the secret's abac_operator value of "620 710" — so the condition would
still not match. This means the correct abac_code value to inject also needs to be
confirmed with the cloud automation team.

---

## Two Possible Fixes — Pending Cloud Automation Decision

### Option A: Fix RolesAnywhere Session Tag Injection (proper long-term fix)

  Two things must be done:

  1. Update the RolesAnywhere profile in Terraform to inject abac_code as a session
     tag when aws_ELK is assumed.

     Profile IDs:
       us-east-2 (primary): c68bd204-2405-4398-85b2-01e65113e35d
       us-east-1 (DR):      a81ce7f6-e974-4784-9c53-d24b8ca2b4ee

  2. Confirm and align the correct abac_code value. The GitHub secret has
     abac_operator: "620 710" — so the injected session tag must be 620 or 710.
     The role currently has abac_code: 118 which does not match. Cloud automation
     needs to advise on the correct value and whether the role tag or the secret
     tags need to be updated to align.

  This is the scalable fix — once session tags are correctly injected, any resource
  tagged with the matching abac_operator/abac_admin/abac_viewer value will
  automatically be accessible without further policy changes.

### Option B: Add GitHub Secret ARN to ElkSecretCustomAccess (quick unblock)

  Add the GitHub secret as a second resource in the ElkSecretCustomAccess policy,
  the same way the logstash secret is handled today:

    {
      "Sid": "GithubSecretAccess",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Effect": "Allow",
      "Resource": "arn:aws:secretsmanager:us-east-2:469620122115:secret:github_hsv_internal/itsma/service_elastic_auto_HSV-SRVK6D"
    }

  No RolesAnywhere changes needed. No session tag dependency. Works immediately.

  Trade-off: this is a hardcoded ARN grant rather than a scalable ABAC pattern.
  Every new secret would require a manual policy update. Cloud automation should
  advise whether this fits the broader IAM strategy or if Option A is preferred.

---

## Validation Plan

Once either fix is applied, verify on the ELK host:

  aws secretsmanager describe-secret \
    --secret-id arn:aws:secretsmanager:us-east-2:469620122115:secret:github_hsv_internal/itsma/service_elastic_auto_HSV-SRVK6D \
    --region us-east-2

  aws secretsmanager get-secret-value \
    --secret-id arn:aws:secretsmanager:us-east-2:469620122115:secret:github_hsv_internal/itsma/service_elastic_auto_HSV-SRVK6D \
    --region us-east-2

  Expected result: both commands succeed without AccessDeniedException.

---

## Current Status

Blocked pending Cloud Automation team decision on preferred remediation path
(Option A vs Option B) and confirmation of correct abac_code value if Option A
is chosen. Message sent to Jesse (Cloud Automation) via Teams with full analysis
and both options outlined.
```
