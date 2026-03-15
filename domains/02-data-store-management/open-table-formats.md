# Open Table Formats and Vector Concepts

Covers transactional data lake formats and vector database concepts.

---

## Open Table Formats

### The Problem
Standard S3 files (Parquet, CSV) are **immutable**. You cannot UPDATE or DELETE individual rows without rewriting the entire file. This is a problem for:
- GDPR "Right to be Forgotten" (delete specific user data)
- Late-arriving data corrections
- Slowly Changing Dimensions (SCD)

### Apache Iceberg
- **AWS's preferred open table format** (deep integration with Athena, EMR, Glue)
- ACID transactions on S3
- Row-level updates and deletes without rewriting all data files
- Schema evolution (add/rename/drop columns)
- Time travel (query historical versions of data)
- Hidden partitioning (partition evolution without rewriting)
- Works with: Athena, EMR, Glue, Redshift

### Apache Hudi
- Similar ACID capabilities to Iceberg
- Copy-on-Write (CoW) and Merge-on-Read (MoR) tables
- Strong for CDC (Change Data Capture) use cases
- Works with: EMR, Glue

### Delta Lake
- Open-source from Databricks
- ACID transactions, time travel
- Works with: EMR (Spark)

### Comparison
| Feature | Iceberg | Hudi | Delta Lake |
|---------|---------|------|------------|
| **AWS Integration** | Best (Athena, Glue, EMR, Redshift) | Good (EMR, Glue) | EMR only |
| **Row-level Ops** | Yes | Yes | Yes |
| **Schema Evolution** | Strong | Moderate | Good |
| **Time Travel** | Yes | Yes | Yes |
| **Exam Relevance** | Highest | Medium | Lower |

---

## Vector Concepts (New for Exam)

### Vector Index Types

**HNSW (Hierarchical Navigable Small Worlds)**
- Graph-based approximate nearest neighbor search
- Fast query performance
- Higher memory usage
- Supported in: Aurora PostgreSQL (pgvector)

**IVF (Inverted File Index)**
- Cluster-based approximate nearest neighbor search
- Lower memory usage than HNSW
- Slightly lower recall
- Good for large-scale vector datasets

### Vectorization in AWS
- **Amazon Bedrock Knowledge Base:** Stores and retrieves vector embeddings
- **Aurora PostgreSQL with pgvector:** Vector similarity search alongside relational data
- **Amazon OpenSearch Service:** Vector search capabilities

---

## Exam Gotchas

- **"Row-level delete/update on S3"** = Open table format (Iceberg is the default answer)
- **"GDPR right to be forgotten in data lake"** = Iceberg/Hudi
- **"Time travel"** or **"query historical data versions"** = Iceberg
- Apache Iceberg has the deepest AWS integration - prefer it in exam answers
- **"Vector similarity search"** = HNSW index type
- Know that vectorization concepts exist (Bedrock, Aurora pgvector) but deep ML knowledge is out of scope
