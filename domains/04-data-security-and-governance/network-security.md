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
