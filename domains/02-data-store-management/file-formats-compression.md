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

### ORC vs Parquet — When to Choose

| Criteria | Parquet | ORC |
|---|---|---|
| Query engine | Athena, Redshift Spectrum, Spark, Glue | Hive on EMR |
| Nested data types | Better support | Limited |
| Predicate pushdown | Row group min/max + Bloom filters | Stripe-level statistics |
| Default choice | Yes — most AWS services prefer Parquet | Only when pipeline is Hive end-to-end |

**Exam signal:** "Hive workload on EMR" → ORC. Everything else (Athena, Redshift Spectrum, Glue, Spark) → Parquet.

### Avro for Schema Evolution
- Schema stored with data
- Supports adding columns (backward compatible)
- Good for streaming pipelines where schema changes over time

---

## Compression

| Algorithm | Compression Ratio | Speed | Splittable | Use Case |
|-----------|-------------------|-------|-----------|----------|
| **Snappy** | Medium | Fast | No (standalone); Yes inside Parquet/ORC | Default for Spark/Glue output |
| **GZIP** | High | Slower | No (standalone); Yes inside Parquet/ORC | Maximum compression, less frequent reads |
| **LZO** | Medium | Fast | No (standalone); Yes with external index | EMR/Hadoop workloads |
| **ZSTD** | High | Fast | No (standalone); Yes inside Parquet/ORC | Good balance of ratio and speed |
| **bzip2** | Very High | Slowest | Yes (natively splittable at block level) | Archival storage |

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

## Parquet/ORC Data Skipping — Min/Max Statistics

Before any data is read, Parquet and ORC store **min/max statistics per column** at the row group (Parquet) or stripe (ORC) level. Query engines use these to skip entire blocks without reading them.

**What is a row group?**

A row group is the horizontal partitioning unit inside a Parquet file. A single Parquet file is divided into one or more row groups (typically 128 MB each). Within each row group, data is stored column by column. ORC uses the equivalent concept called a **stripe**.

**How min/max statistics work:**

For each row group, the Parquet footer stores the minimum and maximum value of every column. When a query contains a filter, the engine checks each row group's min/max before reading:

```
Row group 1: sale_date min=2023-01-01, max=2023-03-31
Row group 2: sale_date min=2023-04-01, max=2023-06-30
Row group 3: sale_date min=2023-07-01, max=2023-09-30

Query: WHERE sale_date = '2023-05-15'
→ Skip row group 1 (max < filter value)
→ Read row group 2 (filter falls within range)
→ Skip row group 3 (min > filter value)
```

**What min/max statistics are good for:**
- Range queries: `WHERE date BETWEEN ...`, `WHERE amount > 1000`
- Equality filters on sorted/clustered columns
- Best when data is written in sorted order on the filter column (sort on write in Spark/Glue)

**What min/max statistics cannot do:**
- Point lookups on high-cardinality unsorted columns — min/max spans the entire range so no rows groups can be skipped (this is where Bloom filters help)

**Exam Pattern:**
- "Athena queries filtering on a date column are slow — optimize without changing query" → Ensure data is written sorted by date so min/max statistics can skip most row groups

---

## Parquet Bloom Filters

Parquet files can embed Bloom filters at the row group level to speed up **point lookup** queries — complementing min/max statistics which handle range queries.

**What is a Bloom filter?**

A Bloom filter is a compact probabilistic data structure stored per column chunk within the Parquet file (the file footer contains pointers to each Bloom filter's location). For each row group, it records which values are present in a specific column. Before reading a row group, query engines check the Bloom filter first:
- If the filter says the value is **DEFINITELY NOT present** → skip the row group entirely (no I/O)
- If the filter says the value **MIGHT be present** → read the row group and check

Result: For high-cardinality point lookups (`WHERE id = 'abc-123'`), most row groups are skipped without being read. Massive I/O reduction on large Parquet files.

**When Bloom Filters Help:**
- High-cardinality columns: `user_id`, `order_id`, `product_sku`, `device_id`
- Point lookups: `WHERE id = specific_value` (exact match, not range)
- Large files with many row groups (the more row groups that can be skipped, the bigger the benefit)

**When Bloom Filters Don't Help:**
- Low-cardinality columns: `country`, `status`, boolean flags (too many matches, filters mostly useless)
- Range queries: `WHERE amount BETWEEN 100 AND 200` (Bloom filters are for exact values only — use min/max statistics for ranges)
- Already well-partitioned data where partition pruning already eliminates most files

**Supported by:**
- **Amazon Athena:** automatically uses Bloom filters when present in Parquet files
- **Apache Spark:** supports reading and writing Bloom filters
- **AWS Glue:** Spark-based jobs support Bloom filters

Write Parquet with Bloom filters in Spark:
```python
df.write \
  .option("parquet.bloom.filter.enabled#user_id", "true") \
  .option("parquet.bloom.filter.expected.ndv#user_id", "1000000") \
  .parquet("s3://my-bucket/data/")
```

**Exam Pattern:**
- "Athena queries on a large Parquet dataset filtering by `user_id` (high cardinality) are slow even with partitioning — how to further reduce data scanned?" → Enable Parquet Bloom filters on the `user_id` column

---

## Exam Gotchas

- **"Optimize Athena query cost"** = Parquet + Snappy + Partitioning
- **"Schema evolution in streaming"** = Avro
- **"Column-level access"** = Parquet/ORC (columnar formats enable reading specific columns only)
- Firehose can convert JSON to Parquet on the fly (needs Glue table)
- Small files problem is a common performance issue - know how to fix it
- Athena charges per TB scanned: Parquet reduces scan size dramatically vs CSV
- **Parquet/ORC min/max statistics skip row groups for range queries** (`WHERE date BETWEEN`, `WHERE amount >`). Writing data sorted on the filter column maximizes skipping. This is separate from and complementary to Bloom filters.
- **Parquet Bloom filters reduce I/O for high-cardinality point lookups** (`WHERE id = value`), but have minimal benefit for low-cardinality columns or range queries. Min/max statistics handle ranges; Bloom filters handle exact-match lookups within a partition.
- **Snappy and GZIP are equally splittable inside Parquet/ORC** — neither is natively splittable as a standalone file. The splittability comes from the Parquet/ORC container. bzip2 is the only common codec that is natively splittable without a container format.
