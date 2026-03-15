# Domain 4: Data Security and Governance (18%)

Covers authentication, authorization, encryption, auditing, and data privacy/governance.

## Task Statements

### Task 4.1: Apply Authentication Mechanisms
- Update VPC security groups
- Create and update IAM groups, roles, endpoints, services
- Create and rotate credentials (Secrets Manager)
- Set up IAM roles for access (Lambda, API Gateway, CLI, CloudFormation)
- Apply IAM policies to roles, endpoints, services (S3 Access Points, PrivateLink)
- Managed vs unmanaged services differences
- SageMaker Unified Studio domains, domain units, and projects

### Task 4.2: Apply Authorization Mechanisms
- Create custom IAM policies
- Store credentials (Secrets Manager, Systems Manager Parameter Store)
- Database user/group/role access (Redshift)
- Manage permissions via Lake Formation (Redshift, EMR, Athena, S3)
- Authorization methods: role-based, tag-based, attribute-based
- Least privilege principle

### Task 4.3: Ensure Data Encryption and Masking
- Data masking and anonymization for compliance
- Encryption keys (KMS)
- Encryption across account boundaries
- Encryption in transit and at rest

### Task 4.4: Prepare Logs for Audit
- CloudTrail for API tracking
- CloudWatch Logs for application logs
- CloudTrail Lake for centralized logging queries
- Analyze logs (Athena, CloudWatch Logs Insights, OpenSearch)
- Integrate services for logging (EMR for large log volumes)

### Task 4.5: Understand Data Privacy and Governance
- Data sharing permissions (Redshift data sharing)
- PII identification (Macie + Lake Formation)
- Data privacy strategies (prevent backup/replication to disallowed regions)
- Configuration change tracking (AWS Config)
- Data sovereignty
- SageMaker Catalog project data access
- Governance frameworks and data sharing patterns

## Study Files

| Topic | File |
|-------|------|
| IAM (Roles, Policies, Cross-Account) | [iam.md](./iam.md) |
| Lake Formation Security (Row/Column/Tag Access) | [lake-formation-security.md](./lake-formation-security.md) |
| Encryption and KMS | [encryption-kms.md](./encryption-kms.md) |
| Network Security (VPC Endpoints, SGs, NACLs) | [network-security.md](./network-security.md) |
| Data Privacy and PII | [data-privacy-pii.md](./data-privacy-pii.md) |
| Audit and Compliance | [audit-compliance.md](./audit-compliance.md) |
