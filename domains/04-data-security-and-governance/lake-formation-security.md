# Lake Formation Security

Fine-grained access control for your data lake.

---

## Why Lake Formation?

Without Lake Formation, securing a data lake requires:
- S3 bucket policies
- IAM policies
- Glue Data Catalog policies
- Creating separate views/datasets per user group

Lake Formation **centralizes all of this** in one place.

---

## Access Control Levels

| Level | Example | How |
|-------|---------|-----|
| **Database** | Analyst sees `analytics_db` | Grant database permission |
| **Table** | Analyst sees `orders` table only | Grant table permission |
| **Column** | Hide `ssn` column from analysts | Column-level permission |
| **Row** | Show only rows where `region='US'` | Row filter expression |
| **Cell** | Combine row + column restrictions | Most granular |

---

## Tag-Based Access Control (TBAC)

Instead of managing permissions per-table/per-column:
1. Assign tags to resources: `sensitivity=high`, `department=finance`
2. Grant access based on tag match: "Finance role can access `department=finance`"
3. New tables with matching tags automatically inherit permissions

### Benefits
- Scales better than individual grants
- New resources automatically get correct permissions via tags
- Easier to audit and manage

---

## Lake Formation + Other Services

| Service | Integration |
|---------|-------------|
| **Athena** | Enforces column/row-level security during queries |
| **Redshift Spectrum** | Enforces Lake Formation permissions on external tables |
| **EMR** | Row/column filtering when reading from catalog |
| **Glue** | Permissions on catalog objects |

---

## Data Sharing

### Redshift Data Sharing
- Share live data between Redshift clusters without copying
- Cross-account and cross-region sharing
- Consumer pays for compute, producer pays for storage

### Lake Formation Cross-Account
- Share data catalog tables/databases across AWS accounts
- RAM (Resource Access Manager) or direct Lake Formation grants
- Consumer uses shared tables in Athena, EMR, etc.

---

## Exam Gotchas

- **"Column-level security"** or **"row-level security"** = Lake Formation
- **"Tag-based access control"** = Lake Formation TBAC
- Lake Formation replaces the need for complex S3 + IAM + Glue policy combinations
- Lake Formation permissions are enforced by Athena, Redshift Spectrum, and EMR
- **"Share data across accounts without copying"** = Redshift Data Sharing or Lake Formation cross-account
