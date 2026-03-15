# AWS Database Migration Service (DMS)

Migrate databases to AWS quickly and securely.

---

## Key Components

### Replication Instance
- EC2 instance that runs migration tasks
- Size correctly for Memory/CPU based on workload
- Runs in a VPC

### Migration Tasks
| Type | Description |
|------|-------------|
| **Full Load** | Migrate all existing data |
| **Full Load + CDC** | Migrate existing data, then capture ongoing changes |
| **CDC Only** | Capture only ongoing changes (assumes initial load done separately) |

### Schema Conversion Tool (SCT)
- **PC Software** (not a cloud service) - runs on your machine
- Used **before** DMS to convert schema (tables, views, stored procedures) from one engine to another
- Required for heterogeneous migrations (e.g., Oracle to Aurora PostgreSQL)
- Not needed for homogeneous migrations (e.g., MySQL to Aurora MySQL)

---

## CDC (Change Data Capture)

Captures ongoing changes (INSERTs, UPDATEs, DELETEs) from source database.

### Source Requirements
- **MySQL:** Binary logging (Binlog) must be enabled
- **PostgreSQL:** Write-Ahead Logging (WAL) must be enabled
- **Oracle:** Supplemental logging must be enabled

---

## Common Use Cases

| Use Case | Pattern |
|----------|---------|
| One-time migration | Move on-prem DB to RDS/Aurora |
| Continuous replication | Replicate production RDS (OLTP) to Redshift (OLAP) or S3 (Data Lake) |
| Data lake ingestion | DMS CDC to S3 for analytics without impacting master DB |
| Cross-region migration | Move databases between AWS regions |

---

## Exam Gotchas

- DMS supports **homogeneous** (Oracle to Oracle) and **heterogeneous** (Oracle to Aurora) migrations
- SCT is needed for heterogeneous migrations to convert schema first
- CDC requires source database-specific configuration (Binlog, WAL, etc.)
- Replication instance must be sized correctly - under-provisioned instances cause migration failures
- DMS can write to S3 in CSV or Parquet format
- For continuous replication to S3/Redshift, use Full Load + CDC task type
