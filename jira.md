## Investigation Summary

Issue: aws_ELK role (assumed via IAM Roles Anywhere) was unable to access the secret
github_hsv_internal/itsma/service_elastic_auto_HSV-SRVK6D despite the role having
the AbacAccess policy attached.

---

## Testing Performed

Two secrets were tested from the ELK host (ms51-22elkalt01) using the aws_ELK role
assumed via RolesAnywhere. Both secrets initially appeared to have identical tags:

  abac_admin:    620 710
  abac_operator: 620 710
  Environment:   shd

Logstash secret (onprem/logstash/automation-9M8cvG)
  Result: SUCCESS ✅ (covered by ElkSecretCustomAccess direct ARN grant — ABAC
  not involved)

GitHub secret (github_hsv_internal/itsma/service_elastic_auto_HSV-SRVK6D)
  Result: AccessDeniedException ❌
  "User: arn:aws:sts::469620122115:assumed-role/aws_ELK/... is not authorized to
  perform: secretsmanager:DescribeSecret... because no identity-based policy
  allows the action"

---

## Root Cause

The aws_ELK IAM role has the tag:
  abac_code: 118

IAM Roles Anywhere injects this role-level tag into the assumed session as a
principal tag (aws:PrincipalTag/abac_code = 118). The AbacAccess policy grants
access via the following ABAC condition:

  aws:ResourceTag/abac_operator  must contain  aws:PrincipalTag/abac_code

The GitHub secret's abac_operator value was "620 710" — which does not contain
"118". The ABAC condition therefore evaluated to false and access was denied.

The logstash secret appeared to work with the same tags, but that was misleading —
it works via a separate direct ARN grant in ElkSecretCustomAccess and is completely
independent of ABAC.

Before fix:
  aws:ResourceTag/abac_operator ("620 710")  contains  aws:PrincipalTag/abac_code ("118")
  ❌ — condition fails, access denied

---

## Fix Applied

Added 118 to the abac_operator and abac_admin tag values on the GitHub secret:

  abac_admin:    620 710 118
  abac_operator: 620 710 118

After fix:
  aws:ResourceTag/abac_operator ("620 710 118")  contains  aws:PrincipalTag/abac_code ("118")
  ✅ — condition passes, access granted

---

## Validation

After updating the secret tags, the following commands succeeded on the ELK host:

  aws secretsmanager describe-secret \
    --secret-id arn:aws:secretsmanager:us-east-2:469620122115:secret:github_hsv_internal/itsma/service_elastic_auto_HSV-SRVK6D \
    --region us-east-2
  Result: SUCCESS ✅

  aws secretsmanager get-secret-value \
    --secret-id arn:aws:secretsmanager:us-east-2:469620122115:secret:github_hsv_internal/itsma/service_elastic_auto_HSV-SRVK6D \
    --region us-east-2
  Result: SUCCESS ✅

---

## Status: RESOLVED ✅

Fix was purely on the secret resource tags — no RolesAnywhere or IAM policy
changes were required.
