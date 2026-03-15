# Domain 2: Data Store Management (26%)

Covers choosing the right data store, designing data models, cataloging schemas, and managing data lifecycles.

## Task Statements

### Task 2.1: Choose a Data Store
- Implement storage services for specific cost and performance requirements (Redshift, EMR, Lake Formation, RDS, DynamoDB, Kinesis, MSK)
- Configure storage for specific access patterns
- Apply storage to appropriate use cases (vector indexing with HNSW, MemoryDB for fast key/value)
- Integrate migration tools (Transfer Family)
- Implement data migration or remote access (Redshift federated queries, materialized views, Spectrum)
- Manage locks to prevent access (Redshift, RDS)
- Manage open table formats (Apache Iceberg)
- Describe vector index types (HNSW, IVF)

### Task 2.2: Understand Data Cataloging Systems
- Use data catalogs to consume data from source
- Build and reference technical data catalogs (Glue Data Catalog, Apache Hive metastore)
- Discover schemas with Glue crawlers
- Synchronize partitions with data catalog
- Create source/target connections (Glue)
- Create and manage business data catalogs (SageMaker Catalog)

### Task 2.3: Manage the Lifecycle of Data
- Load/unload operations between S3 and Redshift
- Manage S3 Lifecycle policies (storage tier transitions, expiration)
- Manage S3 versioning and DynamoDB TTL
- Delete data for business and legal requirements
- Protect data with appropriate resiliency and availability

### Task 2.4: Design Data Models and Schema Evolution
- Design schemas for Redshift, DynamoDB, Lake Formation
- Address changes to data characteristics
- Perform schema conversion (SCT, DMS Schema Conversion)
- Establish data lineage (SageMaker ML Lineage Tracking, SageMaker Catalog)
- Best practices for indexing, partitioning, compression
- Vectorization concepts (Bedrock knowledge base)

## Study Files

| Service/Topic | File |
|--------------|------|
| Amazon S3 Data Lake (Storage Classes, Lifecycle, Partitioning) | [amazon-s3-data-lake.md](./amazon-s3-data-lake.md) |
| Amazon Redshift (Architecture, Spectrum, Distribution/Sort Keys) | [amazon-redshift.md](./amazon-redshift.md) |
| Amazon DynamoDB (Keys, Indexes, DAX, Capacity) | [amazon-dynamodb.md](./amazon-dynamodb.md) |
| Amazon RDS, Aurora, and Other Databases | [amazon-rds-aurora.md](./amazon-rds-aurora.md) |
| AWS Lake Formation + Glue Data Catalog | [aws-lake-formation.md](./aws-lake-formation.md) |
| Data Modeling (Star/Snowflake, OLTP vs OLAP) | [data-modeling.md](./data-modeling.md) |
| File Formats and Compression | [file-formats-compression.md](./file-formats-compression.md) |
| Open Table Formats (Iceberg, Hudi, Delta) | [open-table-formats.md](./open-table-formats.md) |
