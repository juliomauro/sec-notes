# One Wildcard in a Trust Policy

> **Category:** Cloud · IAM · Privilege Escalation  
> **Level:** Advanced  

---

IAM privilege escalation in cloud environments is rarely about a leaked admin key. More often it is about a trust relationship that was designed with good intentions and one operator character that made it wider than intended.

This note is about one of the most common patterns: the `StringLike` condition with a wildcard in a role trust policy.

## How Trust Policies Work

When a role is created in AWS, it has two types of policies attached:

1. **Permission policy** — what the role can do
2. **Trust policy** — who can assume the role

The trust policy controls the `sts:AssumeRole` action. A typical trust policy might say: "only the CI/CD pipeline role can assume me."

```json
{
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::111111111111:role/deploy-prod"},
    "Action": "sts:AssumeRole"
  }]
}
```

This is specific and safe. The problem starts when conditions are used instead of an exact principal match.

## The Vulnerable Pattern

```json
{
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"AWS": "*"},
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringLike": {
        "aws:PrincipalArn": "arn:aws:iam::111111111111:assumed-role/deploy-*/*"
      }
    }
  }]
}
```

The intent: only roles whose name starts with `deploy-` can assume this role.

The problem: `StringLike` with `*` is a pattern match, not an exact match. Any role in the account whose name starts with `deploy-` satisfies this condition — including `deploy-readonly`, `deploy-staging`, `deploy-old-2022`, or any future role created with that prefix for any reason.

If a low-privilege identity can assume any role matching `deploy-*`, it can then use that role to assume the target role.

## The Escalation in Practice

Starting identity: `app-reader` — read-only permissions, no obvious path forward.

Step 1: enumerate trust policies using `iam:GetRole`:

```bash
aws iam get-role --role-name automation-deploy \
  --query 'Role.AssumeRolePolicyDocument'
```

Step 2: identify a role matching the pattern that `app-reader` can assume:

```bash
aws iam list-roles --query 'Roles[?starts_with(RoleName, `deploy-`)].RoleName'
# deploy-readonly
```

Step 3: assume `deploy-readonly` (permitted by its own trust policy):

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::111111111111:role/deploy-readonly \
  --role-session-name pivot
```

Caller ARN is now: `arn:aws:iam::111111111111:assumed-role/deploy-readonly/pivot`

This matches `deploy-*/*`. The condition on `automation-deploy` is satisfied.

Step 4: assume `automation-deploy`:

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::111111111111:role/automation-deploy \
  --role-session-name escalated
```

Full automation permissions acquired. Two API calls from a read-only starting point.

## The Fix

**Use `StringEquals` with a specific ARN:**

```json
"Condition": {
  "StringEquals": {
    "aws:PrincipalArn": "arn:aws:iam::111111111111:assumed-role/deploy-prod/session"
  }
}
```

Or, better — use a named principal directly without conditions:

```json
"Principal": {
  "AWS": "arn:aws:iam::111111111111:role/deploy-prod"
}
```

**Limit `iam:GetRole` and `iam:ListRoles`:**

These permissions allow full enumeration of trust relationships. A low-privilege identity should not be able to read the trust policy of every role in the account.

**Use IAM Access Analyzer:**

AWS IAM Access Analyzer continuously evaluates trust policies and flags external access and overly permissive conditions. Enable it in every account and review its findings regularly.

**Audit with a graph mindset:**

Trust policies cannot be reviewed in isolation. The question is not "who can assume this role?" but "starting from any identity in this account, can someone reach this role through a chain of assumptions?" Tools like PMapper (Principal Mapper) build this graph automatically.

## One Character, One Operator

The difference between a safe trust policy and the vulnerable one above is `StringEquals` vs `StringLike`. One character of difference in the operator name. The wildcard in the ARN pattern becomes inert with `StringEquals` — it matches literally, not as a glob.

This is the class of misconfiguration that does not show up in vulnerability scanners, does not have a CVE, and is invisible to anyone reviewing the policy without understanding how `StringLike` works in this context. It only becomes visible when you think about the policy as part of a permission graph — not as a standalone document.

---

*Related: [Case 05 — Cloud IAM Trust Policy and Role Chaining](https://github.com/juliomauro/sec-writeups/tree/main/cases/05-cloud-iam-trust)*
