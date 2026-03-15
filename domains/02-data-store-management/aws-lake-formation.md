# AWS Lake Formation and Glue Data Catalog

Centralized governance, security, and cataloging for your data lake.

---

## Glue Data Catalog

Central metadata repository for all data assets.

### Components
- **Databases:** Logical grouping of tables
- **Tables:** Schema metadata pointing to data in S3, JDBC, etc.
- **Crawlers:** Auto-discover schema and populate catalog
- **Connections:** Credentials and network info for data sources

### Catalog as Hive Metastore
- Compatible with Apache Hive metastore
- Used by: Athena, Redshift Spectrum, EMR, Glue ETL
- Single source of truth for schema across services

### Partition Sync
- Crawlers can discover new partitions automatically
- `MSCK REPAIR TABLE` in Athena to sync partitions manually
- EventBridge + Lambda can add partitions on S3 object creation

---

## AWS Lake Formation

Specialized security and governance layer built on top of Glue Data Catalog.

### Fine-Grained Access Control
| Level | Description | Example |
|-------|-------------|---------|
| **Database** | Access to entire database | Analyst can see `analytics_db` |
| **Table** | Access to specific tables | Analyst can see `orders` table |
| **Column** | Hide specific columns | Hide `ssn` column from analysts |
| **Row** | Filter rows by condition | Show only rows where `country='US'` |
| **Cell** | Combination of row + column | Most granular |

### Tag-Based Access Control (TBAC)
- Assign tags to tables/columns (e.g., `Confidential`, `PII`)
- Grant access based on tags instead of individual resources
- Scales better than resource-based policies
- Example: Grant `Data Science` role access to tag `Confidential=true`

### Lake Formation Permissions vs IAM
- Lake Formation manages data lake permissions centrally
- Works with: Athena, Redshift, EMR, Glue
- Simplifies what would otherwise require complex S3 bucket policies + IAM policies

### Blueprints
- Pre-built workflows to ingest data into the lake
- Sources: RDS, on-premises databases
- Handles: crawling, ETL, catalog registration

---

## SageMaker Catalog (Business Data Catalog)

- Business-oriented data catalog for discovery
- Data lineage tracking (SageMaker ML Lineage Tracking)
- Domain, domain units, and projects for organizing data assets
- Integrates with Lake Formation for governance

---

## Exam Gotchas

- **Lake Formation** = answer for "fine-grained access control on data lake" or "column/row level security"
- **TBAC** = scalable access control using tags
- Lake Formation simplifies permissions that would otherwise need S3 policies + IAM + Glue policies
- Glue Data Catalog is the **metadata backbone** used by Athena, Spectrum, EMR, Glue
- Without Lake Formation, you'd need separate views/datasets per user group for column filtering
- Lake Formation integrates with **Macie** for PII identification
