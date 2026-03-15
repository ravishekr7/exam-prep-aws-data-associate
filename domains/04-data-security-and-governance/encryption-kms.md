# Encryption and KMS

Protecting data at rest and in transit.

---

## AWS KMS (Key Management Service)

### Key Types
| Type | Description | Cost |
|------|-------------|------|
| **AWS Owned Keys** | Managed by AWS. Simplest | Free |
| **AWS Managed Keys** | Per-service keys (aws/s3, aws/rds) | Free |
| **Customer Managed Keys (CMK)** | You control rotation, policies | $1/month |

### When to Use CMK
- Need audit trail of key usage (CloudTrail)
- Need to share encrypted data **across accounts**
- Need custom key rotation schedule
- Compliance requires customer-controlled keys

### Symmetric vs Asymmetric
- **Symmetric (AES-256):** Same key encrypts/decrypts. Most common for data at rest
- **Asymmetric (RSA):** Public/private key pair. Used for signing or sharing keys externally

---

## S3 Encryption Modes

| Mode | Key Management | Audit Trail | Use Case |
|------|---------------|-------------|----------|
| **SSE-S3** | S3 manages keys | No | Simplest default encryption |
| **SSE-KMS** | KMS manages keys | Yes (CloudTrail) | Compliance, audit needs |
| **SSE-C** | Customer provides key per request | No | Full key control, key never stored by AWS |
| **CSE** | Client-side encryption before upload | N/A | Data encrypted before leaving client |

### SSE-KMS Performance Consideration
- Each encrypt/decrypt = KMS API call
- KMS has API quotas (5,500 - 30,000 requests/sec depending on region)
- High-throughput workloads may hit throttling
- Solution: Use S3 Bucket Keys (reduces KMS calls by up to 99%)

---

## Envelope Encryption

Solves the problem: KMS can only directly encrypt data up to **4 KB**.

### How It Works
1. KMS generates a **Data Key** (plaintext + encrypted copy)
2. AWS SDK uses plaintext Data Key to encrypt your actual data (any size)
3. Plaintext Data Key is discarded from memory
4. Encrypted Data + Encrypted Data Key are stored together
5. To decrypt: KMS decrypts the Data Key, then the Data Key decrypts the data

---

## Encryption in Transit

- **TLS/SSL:** All AWS API calls use HTTPS by default
- **S3:** Enforce with bucket policy condition `aws:SecureTransport`
- **RDS:** Enable SSL/TLS for database connections
- **Redshift:** Enable SSL for JDBC/ODBC connections

---

## Cross-Account Encryption

To share encrypted data across accounts:
1. Use **Customer Managed Key** (not AWS managed)
2. Add the other account's role to the **KMS key policy**
3. Add the other account to the **S3 bucket policy**
4. Both key access and data access are needed

---

## Exam Gotchas

- **"Audit who accessed the encryption key"** = SSE-KMS (CloudTrail logs KMS calls)
- **"Cross-account encrypted data"** = Must use Customer Managed Key (CMK)
- **"KMS throttling"** = Enable S3 Bucket Keys to reduce API calls
- **Envelope Encryption** = how large files are encrypted (Data Key encrypts data, KMS encrypts Data Key)
- SSE-S3 is the simplest but provides no audit trail
- Gateway VPC Endpoints for S3 keep encryption key exchange within AWS network
