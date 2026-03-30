# Amazon RDS, Aurora, and Other Database Services

Relational and specialty database services. Primarily OLTP (Online Transaction Processing).

---

## Amazon Aurora

### Key Features
- **Storage:** Auto-scales 10 GB to 128 TB. Replicates 6 copies across 3 AZs (2 per AZ) — built into the storage layer, not the compute
- **Performance:** Up to 5x MySQL, 3x PostgreSQL performance
- **Compatibility:** MySQL and PostgreSQL compatible

### Endpoints
| Endpoint | Purpose |
|----------|---------|
| **Cluster Endpoint** | Points to Writer (Primary instance) — use for writes |
| **Reader Endpoint** | Load balances across all Read Replicas — use for reads |
| **Custom Endpoint** | Route traffic to specific instance subset (e.g., high-memory instances for analytics queries) |

### Aurora High Availability (Multi-AZ)

Aurora HA works fundamentally differently from RDS Multi-AZ:
- **No passive standby.** Aurora storage is already replicated 6 ways across 3 AZs at the storage layer — the compute layer just needs to redirect
- **Failover = promoting a read replica** to become the new primary (~30 seconds)
- **If no read replica exists**, Aurora provisions a new instance in a different AZ (slower, ~3–5 minutes)
- The Cluster Endpoint automatically redirects to the new primary after failover — no DNS TTL issue

Contrast with RDS Multi-AZ: RDS keeps a passive standby with synchronous block-level replication. Failover requires a DNS change (60–120 seconds). The standby serves no traffic until failover.

### Aurora Global Database
- Low-latency replication across regions (<1 second)
- Use for: Disaster recovery + local reads in other regions
- Promotes secondary region in < 1 minute for failover
- RPO: ~1 second; RTO: < 1 minute

### Aurora Serverless v2
- Scales capacity instantly based on demand
- Fine-grained scaling (0.5 ACU increments)
- Good for variable/unpredictable workloads
- Minimum capacity is 0.5 ACU (cannot scale to zero — that is an Aurora Serverless v1 feature)

### Aurora Backtrack
- Rewinds an Aurora cluster to a prior point in time **without restoring from backup**
- In-place rewind — no new cluster, no data copy, no waiting for snapshot restore
- Configurable backtrack window: up to 72 hours
- **Aurora MySQL only** — not available for Aurora PostgreSQL
- Use case: Accidentally ran `DELETE FROM orders WHERE 1=1` — backtrack to 2 minutes before

Exam pattern: "Accidentally deleted data from Aurora — recover with minimum RTO without a full snapshot restore" → Aurora Backtrack (if Aurora MySQL). For PostgreSQL, use point-in-time restore.

### Aurora with HNSW (Vector Index)
- Aurora PostgreSQL supports **pgvector** extension
- HNSW (Hierarchical Navigable Small Worlds) index for vector similarity search
- Use case: AI/ML applications needing vector storage alongside relational data

### Aurora Exam Patterns
- "Always route writes to the primary, reads to replicas, without changing connection strings" → Cluster Endpoint (writes) + Reader Endpoint (reads)
- "Need a subset of replicas for heavy analytics queries without slowing down other reads" → Custom Endpoint pointing to high-memory instances
- "Cross-region disaster recovery with < 1 second replication lag" → Aurora Global Database
- "Rewind Aurora MySQL to before an accidental mass DELETE — no snapshot restore" → Aurora Backtrack
- "Store vector embeddings alongside relational customer data in one database" → Aurora PostgreSQL with pgvector

---

## Amazon RDS

Standard managed relational databases: PostgreSQL, MySQL, SQL Server, Oracle, MariaDB.

### Read Replicas
- Scale **read** traffic
- **Asynchronous** replication
- Up to 15 replicas (Aurora) or 5 (standard RDS engines)
- Can be cross-region
- Promotion to primary is **manual**

### Multi-AZ (Standard RDS)
- **Disaster Recovery / High Availability**
- **Synchronous** replication to a **passive standby** in a different AZ
- Automatic failover via DNS change (~60–120 seconds)
- NOT for read scaling — standby serves no traffic until failover
- Note: Aurora does not use this passive-standby model (see Aurora HA above)

### Read Replicas vs Multi-AZ
| Feature | Read Replicas | Multi-AZ |
|---------|--------------|----------|
| **Purpose** | Read scaling | High availability |
| **Replication** | Async | Sync |
| **Failover** | Manual promotion | Automatic |
| **Read Traffic** | Yes | No (standby is passive) |
| **Applies to** | RDS + Aurora | Standard RDS (Aurora HA works differently) |

### Automated Backups vs Manual Snapshots

| Feature | Automated Backups | Manual Snapshots |
|---|---|---|
| **Triggered by** | AWS (daily during backup window) | User (on demand) |
| **Retention** | 1–35 days (configurable) | Kept until explicitly deleted |
| **Point-in-time recovery** | Yes — restore to any second within the retention window | No — restores to the snapshot moment only |
| **Deleted with instance?** | Yes (unless retain is enabled) | No — persist independently |
| **Restore creates** | New DB instance | New DB instance |

Key exam trap: Automated backups are deleted when the RDS instance is deleted (unless you enable final snapshot or the "retain automated backups" option). Manual snapshots survive instance deletion indefinitely.

Exam patterns:
- "Restore an RDS database to exactly 14 minutes before the corruption occurred" → Automated backups with point-in-time recovery
- "Keep a database backup for 2 years for compliance — the instance will be deleted after the project ends" → Manual snapshot (automated backups would be deleted with the instance)

---

## RDS Proxy

RDS Proxy is a fully managed database proxy that sits between your application and RDS/Aurora. It pools and reuses database connections, dramatically reducing connection overhead.

**The Problem It Solves:**

Every Lambda invocation opens a new database connection to RDS. At scale (hundreds of concurrent Lambda invocations), this exhausts RDS's connection limit, causing "too many connections" errors and failed queries.

Without RDS Proxy:
```
Lambda functions × concurrency = connections to RDS
Example: 500 concurrent Lambdas × 1 connection each = 500 RDS connections
RDS db.t3.micro max connections (MySQL): ~85 → immediately exhausted
```

With RDS Proxy:
```
Lambda functions → RDS Proxy (connection pool) → small set of actual RDS connections
500 Lambda invocations might only need 10–20 actual RDS connections
```

**Key Features:**

| Feature | Detail |
|---|---|
| Connection pooling | Multiplexes many application connections onto fewer DB connections |
| Automatic failover | Redirects connections to new primary during Multi-AZ failover — faster than DNS propagation |
| IAM authentication | Applications authenticate to Proxy via IAM instead of DB credentials |
| Secrets Manager integration | Proxy retrieves DB credentials from Secrets Manager — no credentials in app code |
| Supported engines | RDS MySQL, RDS PostgreSQL, RDS MariaDB, Aurora MySQL, Aurora PostgreSQL |

**Exam Patterns:**
- "Lambda functions cause 'too many connections' errors on RDS at scale" → Add RDS Proxy between Lambda and RDS
- "Reduce RDS failover time for a Multi-AZ deployment" → RDS Proxy (maintains connections during failover, no DNS TTL wait)
- "Store database credentials securely and rotate them without application code changes" → RDS Proxy + Secrets Manager (Proxy handles rotation transparently)

---

## Aurora Zero-ETL Integration with Redshift

Aurora Zero-ETL is a fully managed integration that replicates data from Aurora into Amazon Redshift in near-real-time — without building or maintaining a custom ETL pipeline.

**How It Works:**
1. Enable Zero-ETL integration on an Aurora cluster (MySQL or PostgreSQL compatible)
2. Target: a Redshift cluster or Redshift Serverless namespace
3. Aurora changes (inserts, updates, deletes) are continuously replicated to Redshift
4. Data is available in Redshift within seconds to minutes of being written to Aurora
5. No S3 staging, no Glue jobs, no DMS — AWS handles the replication internally

**Before Zero-ETL (traditional pattern):**
```
Aurora → DMS → S3 → Glue ETL → Redshift
(multiple services, multiple failure points, minutes to hours of latency)
```

**With Zero-ETL:**
```
Aurora → Redshift (direct, managed by AWS, near-real-time)
```

**Key Constraints:**
- **Aurora MySQL:** Binary logging must be enabled; requires specific DB parameter group settings
- **Aurora PostgreSQL:** Logical replication must be enabled (`rds.logical_replication = 1`); binary logging is not used
- Redshift cluster must have case-sensitive identifiers enabled
- Only selected database objects can be replicated (not all table types supported)
- This is a one-way replication — Redshift is read-only for the replicated data

**Exam Patterns:**
- "Operational data in Aurora needs to be available in Redshift for analytics within minutes with least operational overhead" → Aurora Zero-ETL to Redshift
- "Eliminate the custom ETL pipeline between Aurora and Redshift" → Zero-ETL integration
- "Near-real-time analytics on Aurora transactional data without managing ETL infrastructure" → Zero-ETL

---

## Database Activity Streams

Database Activity Streams provide a near-real-time stream of all database activity (connections, SQL statements, authentication attempts) delivered to Amazon Kinesis Data Streams.

**Purpose:** Compliance auditing — prove exactly who accessed what data, when, and what SQL they executed. Satisfies requirements from PCI DSS, HIPAA, and SOC 2 auditors.

**How It Works:**
```
RDS/Aurora → Database Activity Stream → Kinesis Data Streams → Lambda/Firehose → S3/CloudWatch
```

Every database event generates a stream record:
- **Who:** IAM user or DB user that executed the query
- **When:** Timestamp with millisecond precision
- **What:** Full SQL statement executed (SELECT, INSERT, UPDATE, DELETE, etc.)
- **Result:** Whether the query succeeded or was denied

**Supported Engines:** Aurora MySQL, Aurora PostgreSQL, RDS for Oracle, RDS for PostgreSQL, RDS for SQL Server

**Two Modes:**
| Mode | Description | Risk |
|---|---|---|
| Asynchronous | Activity records written after the DB operation completes | Records can be permanently lost if the DB crashes before the record is flushed to Kinesis — not suitable for zero-gap compliance |
| Synchronous | DB operation waits for activity record to be acknowledged before completing | No record loss; slight performance overhead |

**Key Differences from CloudTrail:**
- CloudTrail logs AWS API calls (who called `StartDBInstance`, `CreateSnapshot`, etc.)
- Database Activity Streams log SQL-level activity inside the database (`SELECT * FROM customers WHERE ssn = ...`)
- Both are needed for full audit coverage

**Exam Patterns:**
- "Regulatory requirement to audit all SQL queries executed against Aurora — including SELECT statements" → Database Activity Streams (CloudTrail does not capture individual SQL queries)
- "Prove that no unauthorized user accessed a specific table last month" → Database Activity Streams → Kinesis → S3, then query with Athena
- "Real-time alerting when a user runs a SELECT on the salary table" → Database Activity Streams → Kinesis → Lambda → SNS alert
- "Compliance requires zero-gap audit trail — no activity records can be missed" → Database Activity Streams in **Synchronous** mode

---

## Other Database Services (In Scope)

### Amazon DocumentDB
- MongoDB-compatible document database
- Managed, scalable
- Use when: migrating MongoDB workloads to AWS

### Amazon Neptune
- Graph database
- Use for: social networks, knowledge graphs, fraud detection, recommendation engines
- Supports: Apache TinkerPop (Gremlin) and SPARQL

### Amazon Keyspaces
- Apache Cassandra-compatible
- Serverless, scalable
- Use when: migrating Cassandra workloads to AWS

### Amazon MemoryDB
- Redis-compatible in-memory database
- Durable (unlike ElastiCache Redis which is a cache, not a database of record)
- Use for: fast key/value access with durability guarantees

---

## Exam Gotchas

- **"High availability"** for RDS = Multi-AZ (NOT read replicas)
- **"Scale reads"** = Read Replicas
- **Aurora Global Database** = cross-region DR with < 1 second replication
- **Aurora Serverless v2** = variable/unpredictable database workloads; cannot scale to zero (v1 can)
- **DocumentDB** = "MongoDB migration" keyword
- **Neptune** = "graph database" or "relationships between entities"
- **Keyspaces** = "Cassandra migration" keyword
- RDS locks: Use `SKIP LOCKED` or `NOWAIT` for managing concurrent access
- **Aurora Multi-AZ ≠ RDS Multi-AZ.** Aurora promotes a read replica (~30 seconds, no passive standby). RDS Multi-AZ flips DNS to a passive standby (~60–120 seconds). If the exam scenario involves Aurora, the failover behavior and timing are different.
- **Aurora Backtrack is Aurora MySQL only.** For fast recovery without a snapshot restore (when you need to rewind minutes, not hours), Backtrack is the answer. For Aurora PostgreSQL, use point-in-time restore instead.
- **Automated backups are deleted when the RDS instance is deleted.** If you need long-term retention after the instance is gone, you need a manual snapshot. Automated backups support point-in-time recovery; manual snapshots do not.
- **RDS Proxy is the answer whenever Lambda + RDS causes connection exhaustion.** The keyword is "too many connections" or "connection pool" in the scenario. RDS Proxy also speeds up Multi-AZ failover because it maintains persistent connections — applications reconnect to the proxy, not the new primary directly.
- **Aurora Zero-ETL replaces DMS + Glue pipelines for Aurora → Redshift scenarios.** When the question asks for "least operational overhead" for near-real-time Aurora-to-Redshift replication, Zero-ETL is always the correct answer over DMS.
- **Database Activity Streams ≠ CloudTrail.** CloudTrail captures AWS API calls (control plane). Database Activity Streams capture SQL-level database activity (data plane). A compliance requirement to audit "which queries accessed sensitive data" requires Database Activity Streams — CloudTrail cannot provide this.
- **Database Activity Streams async mode can lose records on a crash.** For zero-gap compliance auditing, use Synchronous mode — the slight performance cost is the trade-off.
- **Read Replicas ≠ Multi-AZ.** Read Replicas = horizontal read scaling (async replication, manual failover). Multi-AZ = high availability (sync replication, automatic failover). The exam gives both as options — pick based on whether the requirement is "scale reads" or "survive instance failure."
