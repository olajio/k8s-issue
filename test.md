Looking at your policies, I can see a few likely culprits for the access denied errors.

## Why You're Getting Access Denied

### 1. ABAC Tag Matching Is the Core Issue

All three `AbacAccess` statements use **attribute-based access control (ABAC)** — they only grant access when the resource has a matching tag. Specifically:

- `AbacAccessForViewers` requires the resource to have tag `abac_viewer` containing the value of your principal's `abac_code` tag
- `AbacAccessForOperators` requires `abac_operator` tag on the resource
- `AbacAccessForAdmins` requires `abac_admin` tag on the resource

**The most common failure point:** Either your IAM principal (role/user) doesn't have the `abac_code` session tag set, or the resources you're testing against don't have the corresponding `abac_viewer`/`abac_operator` tags.

### 2. How You're Assuming the Role Matters

Looking at `Trustedentities.json`, the role can be assumed via:
- SAML federation (Azure AD or `sts.hedgeservtest.com`) — session tags must come from the IdP SAML assertion
- **IAM Roles Anywhere** — requires the cert CN to match `ROLESANYWHERE_ELK_*` AND the `aws:PrincipalTag/abac_code` must be set

If you're testing with `aws sts assume-role` directly (not through SAML/RolesAnywhere), **the trust policy won't allow it** — there's no `AWS` principal block for direct role assumption.

### 3. `ElkSecretCustomAccess` Is Fine But Narrow

That policy grants direct access to one specific secret ARN without any tag condition, so that part should work independently once the role is assumed correctly.

---

## What to Check / Fix

**Step 1 — Verify your session tags after assuming the role:**
```bash
aws sts get-caller-identity
# Then check what tags are on your session:
aws iam list-role-tags --role-name <your-role-name>
```

**Step 2 — Check if resources have the required ABAC tags.** For example, an SSM parameter you're trying to read needs:
```
abac_viewer = <your-abac_code-value>
# or
abac_operator = <your-abac_code-value>
```

**Step 3 — Simulate the policy with the actual context:**
```bash
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::469620122115:role/<your-role-name> \
  --action-names ssm:GetParameter \
  --resource-arns arn:aws:ssm:us-east-1:469620122115:parameter/<param-name> \
  --context-entries \
    Key=aws:PrincipalTag/abac_code,Type=String,Values=<your-code> \
    Key=aws:ResourceTag/abac_viewer,Type=String,Values=<your-code>
```

---

## Commands to Test the Role Is Working

**1. Confirm role assumption and identity:**
```bash
aws sts get-caller-identity
```

**2. Test the specific secret that `ElkSecretCustomAccess` covers (no tags needed):**
```bash
aws secretsmanager describe-secret \
  --secret-id arn:aws:secretsmanager:us-east-2:469620122115:secret:onprem/logstash/automation-9M8cvG \
  --region us-east-2
```
This should succeed regardless of ABAC since it's a direct ARN grant — if this fails, the role itself isn't being assumed correctly.

**3. Test SSM read on a tagged parameter:**
```bash
aws ssm get-parameter \
  --name /your/parameter/name \
  --region us-east-1
```

**4. Test EC2 describe on a tagged instance:**
```bash
aws ec2 describe-instances \
  --filters "Name=tag:abac_viewer,Values=*<your-abac_code>*" \
  --region us-east-1
```

**5. Use CloudTrail to see the actual denial reason:**
```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=GetParameter \
  --region us-east-1 \
  --query 'Events[?ErrorCode!=`null`].[EventTime,EventName,ErrorCode,ErrorMessage]' \
  --output table
```
The `ErrorMessage` in CloudTrail will tell you exactly which condition failed.

---

## TL;DR Checklist

| Check | What to look for |
|---|---|
| Role assumed via correct method? | Must be SAML or RolesAnywhere — no direct assume-role |
| `abac_code` tag set on your session? | Must come from IdP assertion or RolesAnywhere cert |
| Resources tagged with matching ABAC tags? | `abac_viewer`/`abac_operator` must match your `abac_code` |
| Test the direct-ARN secret first | Isolates role assumption from ABAC issues |
