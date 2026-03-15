# File Formats and Compression

Choosing the right file format and compression has massive impact on query performance and cost.

---

## File Formats

### Columnar Formats (Preferred for Analytics)

| Format | Description | Best For |
|--------|-------------|----------|
| **Apache Parquet** | Column-oriented, efficient compression | Athena, Redshift Spectrum, Glue. Default choice for analytics |
| **Apache ORC** | Optimized Row Columnar | Hive on EMR, similar to Parquet |

**Why columnar?**
- Fetches only needed columns (skip irrelevant data)
- Better compression (similar values in a column)
- Drastically reduces I/O and cost (Athena charges per TB scanned)

### Row-Based Formats

| Format | Description | Best For |
|--------|-------------|----------|
| **CSV** | Simple, human-readable | Data exchange, simple ingestion |
| **JSON** | Nested/semi-structured data | API responses, logs |
| **Apache Avro** | Row-based with schema in file header | Schema evolution, streaming (Kafka/Kinesis) |

### Avro for Schema Evolution
- Schema stored with data
- Supports adding columns (backward compatible)
- Good for streaming pipelines where schema changes over time

---

## Compression

| Algorithm | Compression Ratio | Speed | Splittable | Use Case |
|-----------|-------------------|-------|-----------|----------|
| **Snappy** | Medium | Fast | Yes (with Parquet) | Default for Spark/Glue output |
| **GZIP** | High | Slower | No (unless with Parquet/ORC) | Maximum compression, less frequent reads |
| **LZO** | Medium | Fast | Yes (with index) | EMR/Hadoop workloads |
| **ZSTD** | High | Fast | Yes (with Parquet) | Good balance of ratio and speed |
| **bzip2** | Very High | Slowest | Yes | Archival storage |

### Splittable = Important for Distributed Processing
- Non-splittable files can't be processed in parallel by Spark/EMR
- Parquet and ORC are inherently splittable regardless of compression
- Raw GZIP files are NOT splittable (but GZIP-compressed Parquet is fine)

---

## Format Conversion

### Firehose Format Conversion
- Convert JSON -> Parquet/ORC inline during delivery to S3
- Requires Glue Table definition for schema
- Essential for optimizing Athena query costs

### Glue ETL
- Read any format, write as Parquet/ORC
- Use `ApplyMapping` to reshape schema during conversion

---

## Optimal File Size

- **Too small (KB):** "Small files problem" - excessive S3 LIST overhead, slow metadata operations
  - Fix: Coalesce/Repartition in Spark, `groupFiles` in Glue, Firehose buffering
- **Too large (multi-GB):** Cannot parallelize well
- **Sweet spot:** 128 MB - 1 GB per file for Spark/Athena

---

## Exam Gotchas

- **"Optimize Athena query cost"** = Parquet + Snappy + Partitioning
- **"Schema evolution in streaming"** = Avro
- **"Column-level access"** = Parquet/ORC (columnar formats enable reading specific columns only)
- Firehose can convert JSON to Parquet on the fly (needs Glue table)
- Small files problem is a common performance issue - know how to fix it
- Athena charges per TB scanned: Parquet reduces scan size dramatically vs CSV
