# Amazon RDS, Aurora, and Other Database Services

Relational and specialty database services. Primarily OLTP (Online Transaction Processing).

---

## Amazon Aurora

### Key Features
- **Storage:** Auto-scales 10 GB to 128 TB. Replicates 6 copies across 3 AZs
- **Performance:** Up to 5x MySQL, 3x PostgreSQL performance
- **Compatibility:** MySQL and PostgreSQL compatible

### Endpoints
| Endpoint | Purpose |
|----------|---------|
| **Cluster Endpoint** | Points to Writer (Primary instance) |
| **Reader Endpoint** | Load balances across all Read Replicas |
| **Custom Endpoint** | Route traffic to specific instance subset |

### Aurora Global Database
- Low-latency replication across regions (<1 second)
- Use for: Disaster recovery + local reads in other regions
- Promotes secondary region in < 1 minute for failover

### Aurora Serverless v2
- Scales capacity instantly based on demand
- Fine-grained scaling (0.5 ACU increments)
- Good for variable/unpredictable workloads
- No idle cost if scaled to zero (v1 feature, v2 has minimum)

### Aurora with HNSW (Vector Index)
- Aurora PostgreSQL supports **pgvector** extension
- HNSW (Hierarchical Navigable Small Worlds) index for vector similarity search
- Use case: AI/ML applications needing vector storage alongside relational data

---

## Amazon RDS

Standard managed relational databases: PostgreSQL, MySQL, SQL Server, Oracle, MariaDB.

### Read Replicas
- Scale **read** traffic
- **Asynchronous** replication
- Up to 15 replicas (Aurora) or 5 (other engines)
- Can be cross-region

### Multi-AZ
- **Disaster Recovery / High Availability**
- **Synchronous** replication to standby in different AZ
- Automatic failover (DNS switches to standby)
- NOT for read scaling (standby is passive)

### Read Replicas vs Multi-AZ
| Feature | Read Replicas | Multi-AZ |
|---------|--------------|----------|
| **Purpose** | Read scaling | High availability |
| **Replication** | Async | Sync |
| **Failover** | Manual promotion | Automatic |
| **Read Traffic** | Yes | No (standby is passive) |

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
- Durable (unlike ElastiCache)
- Use for: fast key/value access with durability guarantees

---

## Exam Gotchas

- **"High availability"** for RDS = Multi-AZ (NOT read replicas)
- **"Scale reads"** = Read Replicas
- **Aurora Global Database** = cross-region DR with < 1 second replication
- **Aurora Serverless v2** = variable/unpredictable database workloads
- **DocumentDB** = "MongoDB migration" keyword
- **Neptune** = "graph database" or "relationships between entities"
- **Keyspaces** = "Cassandra migration" keyword
- RDS locks: Use `SKIP LOCKED` or `NOWAIT` for managing concurrent access
