# Data Modeling

Covers OLTP vs OLAP design patterns, schema types, and modeling best practices.

---

## OLTP vs OLAP

| Feature | OLTP (RDS, DynamoDB) | OLAP (Redshift, Athena) |
|---------|---------------------|------------------------|
| **Focus** | Transactions, precise lookups | Analytics, aggregations |
| **Queries** | Simple, return few rows | Complex, huge scans |
| **Data** | Current, volatile | Historical, static |
| **Normalization** | 3NF (avoid redundancy) | Denormalized (Star/Snowflake) |
| **Writes** | Row-by-row | Batch loads |
| **Latency** | Low (ms) | Higher (seconds to minutes) |

---

## Schema Designs for OLAP

### Star Schema
- **Fact table** in the center (transactions, events, measures)
- **Dimension tables** around it (who, what, where, when)
- Simple joins (1 level)
- **Preferred for Redshift**
- Example: `fact_sales` joined to `dim_customer`, `dim_product`, `dim_date`

### Snowflake Schema
- Normalized dimension tables (dimensions have sub-dimensions)
- More complex joins (multiple levels)
- Less storage (reduced redundancy)
- Slower query performance than Star

### Star vs Snowflake for Exam
- Default answer is usually **Star Schema** for Redshift/data warehouse
- Snowflake Schema when storage optimization is priority over query speed

---

## DynamoDB Data Modeling

### Single Table Design
- Store multiple entity types in one table
- Use Partition Key and Sort Key to model relationships
- Example: PK=`CUSTOMER#123`, SK=`ORDER#456`
- Reduces the need for joins (which DynamoDB doesn't support)

### Access Pattern First
- DynamoDB models are designed around **access patterns**, not relationships
- Define all queries before designing the table
- Use GSIs for additional access patterns

---

## Redshift Schema Design

### Distribution + Sort Key Selection
1. Identify most common JOIN columns -> Distribution KEY
2. Identify most common WHERE/ORDER BY columns -> Sort Key
3. Small dimension tables -> ALL distribution
4. Large fact tables -> KEY distribution on join column

---

## Data Mesh vs Data Lake

| Concept | Data Lake | Data Mesh |
|---------|-----------|-----------|
| **Type** | Technical architecture | Organizational pattern |
| **Structure** | Centralized storage (S3) | Decentralized domain ownership |
| **Governance** | Central team manages | Domain teams own their data |
| **Scale Challenge** | Data swamp risk | Coordination across domains |

---

## Exam Gotchas

- **Star Schema** is the default answer for Redshift data modeling
- DynamoDB modeling is **access pattern first**, not relationship first
- **Data Mesh** is organizational (decentralized domains), **Data Lake** is technical (centralized S3)
- Know the difference between normalization (OLTP, 3NF) and denormalization (OLAP, Star)
- Materialized Views in Redshift pre-compute complex aggregations for faster queries
