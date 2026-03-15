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
