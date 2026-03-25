# Network Security

VPC Endpoints, Security Groups, NACLs, and secure connectivity.

---

## VPC Endpoints

Access AWS services privately without traversing the public internet.

### Gateway Endpoints
- **Only for S3 and DynamoDB**
- Free
- Works via Route Table entries
- No DNS changes needed

### Interface Endpoints (PrivateLink)
- For all other services (Kinesis, Glue, Athena, SageMaker, KMS, etc.)
- Uses ENI (Elastic Network Interface) with private IP
- Costs: hourly + data processing
- Requires DNS resolution

### Why Use VPC Endpoints?
- **Security:** Data never leaves AWS network
- **Cost:** Avoids NAT Gateway data processing charges
- **Compliance:** Keep traffic off the public internet

---

## VPC Endpoint Types — Deep Comparison

| Feature | Gateway Endpoint | Interface Endpoint (PrivateLink) |
|---------|-----------------|----------------------------------|
| **Services supported** | **S3 and DynamoDB only** | 100+ AWS services (Glue, Athena, Kinesis, Redshift, KMS, Secrets Manager, etc.) |
| **Cost** | **FREE** | ~$0.01/hour per AZ + $0.01/GB data processed |
| **Network mechanism** | Route table entry — traffic to the service CIDR range is redirected via the endpoint | Creates an **ENI** in your subnet with a private IP address |
| **DNS changes** | No DNS changes — works via route table prefix list | Creates private DNS names (e.g., `kinesis.us-east-1.amazonaws.com` resolves to the ENI's private IP) |
| **Security group** | Not applicable — controlled via endpoint policy and route table | **Has its own security group** — control which resources in your VPC can reach it |
| **Availability Zone** | Regional — single endpoint serves all AZs | Must create one per AZ for high availability |
| **Endpoint policy** | Yes — restrict which S3 buckets or DynamoDB tables are accessible | Yes — restrict which API calls are allowed |

### When You Need Interface Endpoint (Not Gateway)
If your workload runs in a **private subnet** (no internet access, no NAT Gateway) and needs to reach any of these services:
- AWS Glue API (for job management, catalog API calls)
- Amazon Kinesis Data Streams
- Amazon Athena
- Amazon Redshift (for JDBC connections from within VPC)
- AWS Secrets Manager (for reading secrets without internet)
- AWS KMS (for encryption operations)
- Amazon CloudWatch Logs (for writing logs from private instances)

**Exam pattern:** "Glue job in a private subnet fails to call the Glue API to update catalog" → Missing **Interface Endpoint for Glue API**. The Gateway Endpoint (S3/DynamoDB only) does not help here.

---

## VPC Endpoint Policy

A VPC endpoint policy is a **resource-based policy attached to the endpoint itself** — an additional access control layer beyond IAM policies.

### Key Point: Defense in Depth
Even if a user or role has IAM permission to access an S3 bucket, if the endpoint policy does not allow it, the request is denied when made through the endpoint. This enables **data perimeter** controls — restricting which AWS resources can be reached from within your VPC.

### Use Case: Prevent Data Exfiltration to Personal Buckets
Prevent EC2 instances or Glue jobs in your corporate VPC from uploading data to unauthorized S3 buckets (e.g., a developer's personal bucket):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:ListBucket"],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceOrgID": "o-exampleorgid123"
        }
      }
    }
  ]
}
```

With this policy, S3 requests through the endpoint are **only allowed to S3 buckets owned by accounts in your AWS Organization**. Requests to buckets in personal AWS accounts (outside your org) are denied — even if the user's IAM policy technically allows it.

### Additional Use Cases
- Allow only specific S3 buckets: `"Resource": ["arn:aws:s3:::company-data-*", "arn:aws:s3:::company-data-*/*"]`
- Restrict DynamoDB access to specific tables
- Allow read-only access from the endpoint while full access is available via other paths

---

## Security Groups vs NACLs

| Feature | Security Group | NACL |
|---------|---------------|------|
| **Level** | Instance (EC2, RDS, ENI) | Subnet |
| **Stateful?** | Yes (return traffic auto-allowed) | No (must allow both directions) |
| **Rules** | Allow only | Allow and Deny |
| **Default** | Deny all inbound, allow all outbound | Allow all |
| **Evaluation** | All rules evaluated | Rules evaluated in number order |

### Common Pattern for Data Engineering
- Security Group on Glue connection: Allow outbound to RDS on port 3306
- Security Group on RDS: Allow inbound from Glue security group
- This enables Glue to connect to RDS within the VPC

---

## Glue in VPC

- Glue jobs that access VPC resources (RDS, Redshift) need:
  1. VPC, Subnet, Security Group configured in the **Glue Connection**
  2. NAT Gateway or VPC Endpoint for accessing S3 (Glue in VPC loses internet access)
  3. S3 Gateway Endpoint (free) is the recommended approach

---

## PrivateLink for Data Services

### Kinesis Data Streams
Without a VPC endpoint, producers and consumers in a private subnet must route through a NAT Gateway to reach Kinesis. NAT Gateway adds cost (~$0.045/GB) and a potential bottleneck.

**Solution:** Create an **Interface Endpoint for Kinesis Data Streams** in each AZ where producers/consumers run.
- Endpoint DNS name: `kinesis.us-east-1.amazonaws.com` resolves to the endpoint's private IP
- Traffic stays entirely within the AWS network — no NAT Gateway needed
- Reduces NAT Gateway costs, improves security posture

### AWS Glue
Glue jobs running in a VPC need access to multiple endpoints:
1. **S3 Gateway Endpoint (free):** For reading source data and writing output
2. **Glue API Interface Endpoint:** For Glue job management, catalog reads, job heartbeats
3. **CloudWatch Logs Interface Endpoint:** For writing job logs

```
Missing any of these = Glue job fails with connectivity errors
```

**Common exam scenario:** "A Glue ETL job runs in a private subnet with no internet gateway. It fails with a timeout when trying to update the Glue catalog." → The job needs an **Interface Endpoint for the Glue API** to call catalog APIs without internet access.

### Amazon Athena
For private Athena queries (submitting queries from a Lambda or EC2 in a private subnet):
1. **Athena Interface Endpoint:** `athena.us-east-1.amazonaws.com`
2. **S3 Gateway Endpoint:** Athena reads data from S3 and writes query results to an S3 bucket
3. **Glue Interface Endpoint (optional):** If Athena needs to access the Glue catalog API for schema resolution

---

## AWS PrivateLink

- Expose your services to other VPCs/accounts privately
- No VPC peering, no internet gateway needed
- Service provider creates NLB + VPC Endpoint Service
- Consumer creates Interface Endpoint

---

## Exam Gotchas

- **"S3 and DynamoDB VPC access"** = Gateway Endpoint (free)
- **"Keep traffic private"** = VPC Endpoints
- **"Reduce NAT Gateway costs"** = VPC Endpoints
- Security Groups are **stateful** (most common exam distinction)
- NACLs are **stateless** (must explicitly allow return traffic)
- Glue in VPC needs NAT Gateway or S3 VPC Endpoint to access S3
- S3 Access Points can restrict access to specific VPCs
