# IAM (Identity and Access Management)

Controls WHO can do WHAT on WHICH resources in AWS.

---

## Core Concepts

| Concept | Description |
|---------|-------------|
| **User** | Person/service with long-term credentials. Avoid for applications |
| **Group** | Collection of users. Attach policies to groups, not individual users |
| **Role** | Temporary credentials. Best practice for services (EC2, Glue, Lambda) |
| **Policy** | JSON document defining permissions |

---

## Policy Structure

```json
{
  "Effect": "Allow",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::my-bucket/*",
  "Condition": {
    "StringEquals": {
      "s3:prefix": "data/"
    }
  }
}
```

### Key Rules
- **Explicit Deny always wins** over Allow
- Default is Deny (if no policy grants access, access is denied)
- Policies can be identity-based (attached to user/role) or resource-based (attached to S3 bucket, KMS key)

---

## IAM Policy Evaluation Logic

> **The exam tests this heavily. Know the evaluation order and the "explicit deny wins" rule cold.**

### Evaluation Order (Decision Tree)

```
Request comes in
       │
       ▼
┌─────────────────────────────────────────┐
│ 1. Is there an EXPLICIT DENY anywhere?  │ ──YES──▶ DENY (game over, nothing overrides this)
│    (in ANY policy in the chain)         │
└─────────────────────────────────────────┘
       │ NO
       ▼
┌─────────────────────────────────────────┐
│ 2. Is there an SCP (Service Control     │ ──DENY──▶ DENY
│    Policy) that denies this action?     │
└─────────────────────────────────────────┘
       │ ALLOW or N/A
       ▼
┌─────────────────────────────────────────┐
│ 3. Is there a resource-based policy     │ ──YES──▶ ALLOW (for same-account access)
│    (S3 bucket policy, KMS key policy)   │         Cross-account: BOTH resource policy
│    that allows this action?             │         AND identity policy must allow
└─────────────────────────────────────────┘
       │ NO
       ▼
┌─────────────────────────────────────────┐
│ 4. Is there a Permission Boundary set?  │ ──YES──▶ Effective permissions = intersection
│    Does it allow this action?           │         of boundary AND identity policy
└─────────────────────────────────────────┘
       │ NO boundary or boundary allows
       ▼
┌─────────────────────────────────────────┐
│ 5. Does an identity-based policy        │ ──YES──▶ ALLOW
│    (attached to user/role) allow this?  │
└─────────────────────────────────────────┘
       │ NO
       ▼
    IMPLICIT DENY (default — no policy granted access)
```

### The Cardinal Rule
**An explicit Deny in ANY policy at ANY level in the chain wins.** There is no way to override an explicit Deny — not even with `AdministratorAccess`. This is why explicit Deny is used in SCPs, permission boundaries, and endpoint policies to create hard guardrails.

---

## Permission Boundaries

### What It Is
A permission boundary is an IAM managed policy that you attach to a user or role to set the **maximum permissions** that entity can have. Effective permissions are the **intersection** of the identity policy AND the boundary — you only get permissions that BOTH grant.

```
Effective Permissions = Identity Policy ∩ Permission Boundary
```

### Worked Example
```
Developer's identity policy: AdministratorAccess (all AWS actions allowed)
Permission boundary policy:   Allow: s3:*, dynamodb:*

Effective permissions = intersection = s3:* and dynamodb:* only
→ Developer CANNOT create EC2 instances, create IAM roles with elevated permissions,
  or access Redshift — even though AdministratorAccess technically allows all of this.
```

### Use Case: Delegated Administration Without Privilege Escalation
Allow developers to create IAM roles for their microservices, but prevent them from creating roles with more permissions than they themselves have.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["iam:CreateRole", "iam:AttachRolePolicy"],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "iam:PermissionsBoundary": "arn:aws:iam::123456789:policy/DeveloperBoundary"
        }
      }
    }
  ]
}
```

With this policy, developers can create roles — but only if they attach the `DeveloperBoundary` permission boundary to the new role. The new role cannot exceed developer-level permissions.

**Exam pattern:** "Prevent developers from creating IAM roles that escalate their own privileges" → Permission Boundaries.

---

## Cross-Account Access Pattern

### Assume Role (Most Common)
```
Account A (requestor) has: IAM role with sts:AssumeRole permission to Account B role
Account B (target) has: IAM role with trust policy allowing Account A's role

Flow:
1. Account A role calls sts:AssumeRole for Account B role ARN
2. AWS STS verifies:
   - Account A's identity policy allows sts:AssumeRole on Account B's role ✓
   - Account B's role trust policy allows Account A's role as principal ✓
3. STS returns temporary credentials (access key, secret, session token)
4. Account A uses temporary credentials to act within Account B
```

### For S3 Cross-Account Access (Both Policies Required)
Accessing an S3 bucket in Account B from Account A requires **both**:
1. **Account A IAM policy:** Allow `s3:GetObject` on `arn:aws:s3:::account-b-bucket/*`
2. **Account B S3 Bucket Policy:** Allow Account A's role as principal

```json
// Account B's S3 bucket policy (required)
{
  "Effect": "Allow",
  "Principal": {"AWS": "arn:aws:iam::ACCOUNT_A_ID:role/ETLRole"},
  "Action": ["s3:GetObject", "s3:ListBucket"],
  "Resource": [
    "arn:aws:s3:::account-b-data-bucket",
    "arn:aws:s3:::account-b-data-bucket/*"
  ]
}
```

If only one side grants access, the request is denied. Both must allow.

### External ID: Confused Deputy Prevention
When granting a third-party service (e.g., a SaaS analytics tool) access to your AWS account via an IAM role, use an External ID to prevent the confused deputy problem.

```json
// Trust policy with External ID
{
  "Effect": "Allow",
  "Principal": {"AWS": "arn:aws:iam::THIRD_PARTY_ACCOUNT:root"},
  "Action": "sts:AssumeRole",
  "Condition": {
    "StringEquals": {
      "sts:ExternalId": "unique-external-id-provided-by-third-party"
    }
  }
}
```

The external ID ensures that even if a malicious third party knows your role ARN, they cannot assume it without also knowing the unique external ID you share only with your legitimate third-party vendor.

---

## Cross-Account Access

### Option 1: Resource Policy
Add bucket policy in Account B allowing Account A's role:
```json
{
  "Effect": "Allow",
  "Principal": {"AWS": "arn:aws:iam::ACCOUNT_A:role/MyRole"},
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::account-b-bucket/*"
}
```

### Option 2: Assume Role
Account A assumes a role in Account B (role has permissions in Account B).

---

## Service-Linked Roles

- Pre-defined roles for AWS services
- Cannot modify permissions
- Examples: `AWSServiceRoleForEMR`, `AWSGlueServiceRole`

---

## Credential Management

### AWS Secrets Manager
- Store and rotate secrets (database passwords, API keys)
- Automatic rotation with Lambda
- Cross-account sharing
- Audit access via CloudTrail

### Systems Manager Parameter Store
- Store configuration data and secrets
- Free tier available (standard parameters)
- Less features than Secrets Manager (no native rotation)
- Use for: non-sensitive config, simple secrets

### Secrets Manager vs Parameter Store
| Feature | Secrets Manager | Parameter Store |
|---------|----------------|-----------------|
| **Rotation** | Built-in automatic | Manual (Lambda needed) |
| **Cost** | $0.40/secret/month | Free tier available |
| **Cross-Account** | Yes | Limited |
| **Best For** | Database passwords, API keys | Configuration values, simple secrets |

---

## Exam Gotchas

- **Always use Roles** for services (never embed credentials)
- **Least privilege:** Only grant permissions that are needed
- **Explicit Deny** overrides everything
- **Cross-account:** Resource policy OR assume role (know both patterns)
- **"Rotate credentials automatically"** = Secrets Manager
- **"Store configuration parameters"** = Parameter Store
- S3 Access Points provide per-application access to shared datasets
- **IAM policy conditions commonly tested:**
  - `aws:RequestedRegion` — restrict to specific regions
  - `s3:prefix` — restrict to specific S3 path prefixes
  - `aws:SourceIp` — restrict to specific IP ranges or CIDR blocks
  - `aws:PrincipalOrgID` — restrict to principals within your AWS Organization
- **NotAction vs Deny:** `NotAction` is NOT a Deny. `"Effect": "Allow", "NotAction": ["iam:*"]` means "allow everything EXCEPT IAM actions" — but other policies can still explicitly allow those IAM actions. This is different from an explicit Deny, which cannot be overridden.
- **Service Control Policies (SCPs):** Apply to an entire AWS Organizations account or OU. SCPs cannot **grant** permissions — they can only **restrict** the maximum permissions available. Even the root user of an account is bound by SCPs. Commonly used to: prevent disabling CloudTrail, restrict to specific regions, prevent leaving the organization.
