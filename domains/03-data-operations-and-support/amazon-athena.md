# Amazon Athena

Serverless interactive query service to analyze data in S3 using standard SQL.

---

## Key Features

- **Serverless:** No infrastructure to manage
- **Pay per query:** Charged per TB of data scanned
- **SQL Engine:** Based on Presto/Trino
- **Data Sources:** S3 (primary), 25+ federated data source connectors
- **Formats:** CSV, JSON, Parquet, ORC, Avro
- **Catalog:** Uses Glue Data Catalog for schema

---

## Cost Optimization

### Reduce Data Scanned = Reduce Cost
1. **Use columnar formats (Parquet/ORC):** Only read needed columns
2. **Partition data:** Skip irrelevant partitions (partition pruning)
3. **Compress data:** Snappy or GZIP with Parquet
4. **Use partition projection:** Avoid crawling, compute partitions at query time
5. **Use LIMIT:** Only return needed rows

### Pricing
- $5 per TB of data scanned
- Cancelled queries charged for data already scanned
- DDL (CREATE TABLE, ALTER) and failed queries = no charge

---

## Athena Federated Query

- Query data in sources beyond S3 (RDS, DynamoDB, Redshift, CloudWatch Logs, etc.)
- Uses Lambda-based data source connectors
- Enables "query anywhere" without moving data

---

## Athena Notebooks (Apache Spark)

- Run Spark code interactively in Athena
- Jupyter notebook interface
- Use for exploratory data analysis with Spark on S3 data

---

## CTAS (Create Table As Select)

- Create new table from query results
- Useful for: converting formats, creating aggregated datasets
- Example: `CREATE TABLE new_table WITH (format='PARQUET') AS SELECT * FROM csv_table`

---

## Exam Gotchas

- **"Serverless SQL on S3"** = Athena (always)
- **"Ad-hoc queries"** or **"infrequent analysis"** = Athena (cost-effective pay-per-query)
- **"Optimize Athena cost"** = Parquet + Partitioning + Compression
- Athena is NOT ideal for complex joins or frequent dashboards (use Redshift)
- **Partition Projection** eliminates need for Glue Crawlers when partition scheme is known
- CTAS is useful for one-time format conversions
- Athena can analyze CloudTrail logs, VPC Flow Logs, ELB logs stored in S3
