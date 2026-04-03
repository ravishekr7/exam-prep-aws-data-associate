# DEA-C01 Practice Questions

31 scenario-based questions covering all 4 exam domains. Each question mirrors the style and difficulty of actual DEA-C01 exam questions — testing judgment and trade-offs, not memorization.

---

## Domain 1: Data Ingestion and Transformation

---

### Q1 — Streaming Pipeline Design
**Domain 1 | Key Services: Kinesis, Flink, Lambda**

A financial services company receives 8 MB/s of market data from 12 stock exchanges. Three separate downstream teams consume this data: an algorithmic trading team needs sub-100ms latency and 2 MB/s throughput, a risk analytics team needs 2 MB/s with 200ms acceptable latency, and a compliance archiving team needs to write all data to S3 within 60 seconds. The architecture must minimize cost while meeting each team's SLA.

**A)** Use a single Kinesis Data Stream with 4 shards. All three teams use standard GetRecords polling.

**B)** Use a single Kinesis Data Stream with 4 shards and enable Enhanced Fan-Out for the algorithmic trading and risk analytics consumers. Use Kinesis Firehose as a separate consumer for S3 archiving.

**C)** Use Amazon MSK with 8 partitions. All three teams use separate consumer groups.

**D)** Use Kinesis Firehose with a 60-second buffer. Configure three separate Firehose delivery streams, one per team.

**✅ Correct Answer: B**

**Explanation:** With 8 MB/s ingest and the write limit of 1 MB/s per shard, you need ceil(8/1) = 8 shards. Wait — re-read: with read requirements at 2 MB/s per consumer, shared mode gives 2 MB/s total across all consumers (0.67 MB/s each). Enhanced Fan-Out gives each consumer their own 2 MB/s per shard. 4 shards provide 8 MB/s write capacity (4 × 2 = 8) and 4 × 2 = 8 MB/s read per EFO consumer. Firehose as an additional consumer handles S3 archiving without consuming the stream's shared read throughput. **A** fails because 4 shards only provide 8 MB/s read total shared across all consumers. **C** would work but adds unnecessary Kafka management overhead. **D** fails because Firehose cannot serve the sub-100ms requirement and doesn't support custom consumer logic.

---

### Q2 — Glue vs EMR Selection
**Domain 1 | Key Services: Glue, EMR**

A data engineering team runs nightly ETL jobs that process 50 TB of clickstream data stored in S3 as JSON. The jobs perform complex joins across 8 tables, apply custom Python UDFs for session reconstruction, and output Parquet to S3. The jobs run for 4 hours nightly and have been running on Glue for 3 months. The team's cloud cost report shows ETL costs have risen to $15,000/month and the CFO wants a 60% reduction within 90 days. The team has strong Spark expertise but no DevOps capacity for cluster management.

**A)** Migrate to EMR on EC2 with On-Demand m5.4xlarge instances for core nodes and r5.4xlarge Spot instances for task nodes.

**B)** Migrate to EMR Serverless, which is cheaper than Glue and requires no cluster management.

**C)** Stay on Glue but switch from G.2X to G.1X workers to reduce DPU costs.

**D)** Rewrite UDFs as native Spark SQL functions and right-size the Glue job, then evaluate EMR if savings are insufficient.

**✅ Correct Answer: D**

**Explanation:** Before migrating platforms, the team should optimize within the current platform. Python UDFs in Glue are 10-100× slower than native Spark functions — eliminating them could reduce job runtime from 4 hours to <1 hour, dramatically reducing DPU costs. Right-sizing (using fewer workers if the job is I/O bound, not CPU bound) is also necessary. If optimization still doesn't achieve 60% savings, then EMR migration makes sense. **A** adds cluster management overhead the team explicitly cannot handle. **B** is cheaper than Glue but not necessarily 60% cheaper without optimization. **C** alone is unlikely to achieve 60% savings and may increase runtime if jobs are memory-bound.

---

### Q3 — Shard Calculation Under Exam Conditions
**Domain 1 | Key Services: Kinesis**

A logistics company's IoT tracking system will send GPS coordinates from 2 million delivery vehicles. Each vehicle sends one 500-byte record every 5 seconds. A fleet management dashboard reads all data with a 1-second refresh. A machine learning pipeline also reads the full stream for real-time route optimization. What is the minimum number of Kinesis Data Streams shards required?

**A)** 1 shard

**B)** 200 shards

**C)** 400 shards

**D)** 800 shards

**✅ Correct Answer: C**

**Explanation:** Calculate each constraint:
- **Records/sec:** 2,000,000 vehicles ÷ 5 sec = 400,000 records/sec → ceil(400,000 / 1,000) = **400 shards**
- **Write MB/s:** 400,000 records × 500 bytes = 200 MB/s → ceil(200 / 1) = **200 shards**
- **Read MB/s:** 2 consumers × 200 MB/s = 400 MB/s total read → ceil(400 / 2) = **200 shards** (standard mode); OR if using EFO, each consumer gets 2 MB/s per shard, so 200 shards satisfy each consumer independently

The binding constraint is records per second: **400 shards**. Without Enhanced Fan-Out, the read constraint would be 200 shards, which doesn't bind. Without EFO (standard), total read is 200 MB/s shared across consumers → 100 MB/s per consumer → need ceil(200/2) = 100 shards. So the records constraint of 400 is still binding. Answer C.

---

### Q4 — DMS and CDC
**Domain 1 | Key Services: DMS, Kinesis, Glue**

A healthcare company is migrating a 2 TB PostgreSQL database on-premises to AWS. During migration, the source database must remain operational and all changes (INSERT, UPDATE, DELETE) must be captured and replicated to the AWS target within 30 seconds. The target will be an Amazon Aurora PostgreSQL instance. After migration is complete, the CDC stream should feed an S3-based data lake for analytics.

**A)** Use AWS DMS Full Load mode to copy all data, then set up logical replication from PostgreSQL to Aurora.

**B)** Use AWS DMS with Full Load + CDC mode. Set the target as Aurora PostgreSQL for the migration. After cutover, add a second DMS task to also write CDC events to Kinesis Data Streams, which feeds a Glue Streaming ETL job writing to S3.

**C)** Use AWS Database Migration Service with Full Load to Aurora, then use AWS Glue to run a daily incremental sync using a timestamp watermark column.

**D)** Use AWS DataSync to sync all files, then set up a DMS CDC task.

**✅ Correct Answer: B**

**Explanation:** DMS Full Load + CDC is the purpose-built solution for this exact pattern. It performs a full copy while tracking ongoing changes in the transaction log, then seamlessly transitions to CDC mode — ensuring zero data loss during migration with < 30 second lag. Adding a second DMS task for S3/analytics is the standard multi-target pattern. **A** requires manual setup of PostgreSQL logical replication, which is complex and risky during migration. **C** fails the 30-second requirement — daily sync has 24-hour lag. **D** DataSync is for file system/S3 sync, not database CDC.

---

### Q5 — Step Functions vs MWAA
**Domain 1 | Key Services: MWAA, Step Functions**

A data platform team at a retail company manages 300 Apache Airflow DAGs for their data pipeline. Each DAG has complex dependencies between 20-50 tasks, uses custom Python operators for vendor API integrations, and requires backfilling when new data sources are onboarded. The company is moving to AWS and wants to keep operational overhead low. Which orchestration approach should they use?

**A)** Migrate all DAGs to AWS Step Functions Standard Workflows, converting Python operators to Lambda functions.

**B)** Use Amazon MWAA to host the existing Airflow DAGs with minimal changes.

**C)** Use AWS Step Functions Express Workflows for high-frequency DAGs and Standard Workflows for long-running DAGs.

**D)** Use Amazon EventBridge Scheduler to trigger individual Lambda functions for each task, using SQS for task dependencies.

**✅ Correct Answer: B**

**Explanation:** With 300 existing Airflow DAGs, complex Python operators, and a backfill requirement, MWAA is the clear answer. Lift-and-shift of existing DAGs minimizes migration effort. Backfill is a native Airflow concept with no equivalent in Step Functions. **A** would require rewriting 300 DAGs entirely into ASL (Amazon States Language) and converting all Python operators to Lambda — a massive, risky project with no clear benefit. **C** is incorrect because the question involves existing Airflow DAGs, not new workflows. **D** would require building a custom task dependency system from scratch, which is exactly what Airflow already provides.

---

### Q6 — Glue Streaming ETL Configuration
**Domain 1 | Key Services: Glue, Kinesis**

A data engineering team has a Glue Streaming ETL job that reads from Kinesis Data Streams, applies transformations, and writes enriched records to S3. After a job failure, the team restarts the job but notices it starts reprocessing data from the very beginning of the stream (TRIM_HORIZON). This causes duplicate records in S3 and breaks downstream analytics. What should the team configure to ensure the job resumes from where it left off after a restart?

**A)** Set the Kinesis starting position to LATEST instead of TRIM_HORIZON.

**B)** Configure a `checkpointLocation` in the Glue streaming job options pointing to an S3 path. The job will automatically save its position and resume from the checkpoint after restart.

**C)** Increase the Kinesis stream retention period to 7 days so the job can always replay from the last processed record.

**D)** Use Kinesis Enhanced Fan-Out to get a dedicated push connection that maintains position automatically.

**✅ Correct Answer: B**

**Explanation:** The `checkpointLocation` is the mechanism Glue Streaming ETL uses to persist its position in the stream to S3. When the job restarts, it reads from the checkpoint — not from the beginning or LATEST. **A** setting LATEST would cause the job to miss all records written while it was down — data loss, not duplicate prevention. **C** longer retention helps with replay but doesn't solve the resume-from-last-position problem. **D** EFO is about throughput per consumer, not position tracking.

---

### Q7 — Lambda vs Flink for Stream Processing
**Domain 1 | Key Services: Lambda, Flink, Kinesis**

An e-commerce company wants to detect when a customer's session on their website goes inactive (no events for 15 minutes) and trigger an abandoned cart email. Sessions can span millions of concurrent users. The detection must happen within 30 seconds of the session timing out. Which approach is most appropriate?

**A)** AWS Lambda triggered by Kinesis Data Streams. Lambda stores the last event time per session in DynamoDB and checks on each new event whether 15 minutes have elapsed.

**B)** Managed Service for Apache Flink with a session window keyed by customer_id. Flink's built-in session gap trigger fires when no events arrive for 15 minutes. Output the session close event to Kinesis → Lambda → SES for email.

**C)** Amazon Kinesis Firehose with a 15-minute buffer interval. Records that arrive in the buffer represent active sessions; those not received represent abandoned carts.

**D)** AWS Glue Streaming ETL with a 15-minute windowSize. After each window, identify sessions with no activity and trigger SNS.

**✅ Correct Answer: B**

**Explanation:** Session windows with a gap trigger are a native Flink concept. Flink maintains distributed state per customer_id across millions of concurrent sessions, automatically fires when no event arrives within the gap, and outputs the result — all without external storage. **A** would work in theory but requires every Lambda invocation to update DynamoDB, creating massive write pressure. More critically, Lambda cannot trigger on inactivity — it only fires when a record arrives. You'd need a separate scheduled Lambda scanning DynamoDB for timed-out sessions. **C** Firehose doesn't understand session semantics. **D** Glue Streaming uses micro-batches, not event-driven session windows — it would process records in 15-minute chunks, not trigger on inactivity.

---

### Q8 — EMR Deployment Mode Selection
**Domain 1 | Key Services: EMR**

A media streaming company runs Apache Spark jobs to process video encoding metadata. The jobs run unpredictably — sometimes 5 jobs per day, sometimes 50 — each lasting 2-4 hours and processing 200 GB of data. The data engineering team has two engineers and no dedicated infrastructure team. Cost optimization and simplicity are the primary requirements. Which EMR deployment option is most appropriate?

**A)** EMR on EC2 with a persistent cluster, Reserved Instances for core nodes, and Spot Instances for task nodes.

**B)** EMR on EKS using an existing Kubernetes cluster shared with the application team.

**C)** EMR Serverless, which automatically provisions resources for each job and charges only for compute used.

**D)** AWS Glue with G.2X workers, since Glue is serverless and handles Spark natively.

**✅ Correct Answer: C**

**Explanation:** EMR Serverless is purpose-built for variable/bursty workloads with no infrastructure management. With 2 engineers, the team cannot manage clusters (rules out A). The jobs are Spark-based and need EMR's framework flexibility (rules out D for jobs requiring specific Spark configs or frameworks). EMR on EKS requires Kubernetes expertise and a shared cluster (rules out B — adds complexity). EMR Serverless: zero idle cost, auto-scales per job, no cluster lifecycle management. **A** would waste money on idle cluster time during low-volume days. **D** Glue might work but if the jobs use custom JARs or specific Spark configurations not supported by Glue, this won't work.

---

### Q9 — AppFlow vs DMS vs Glue
**Domain 1 | Key Services: AppFlow, DMS**

A marketing operations team needs to sync customer records from Salesforce CRM into Amazon S3 every hour. The data should be stored in Parquet format. The team has no data engineering resources — they need a solution that requires no code or pipeline development. Which service should they use?

**A)** AWS Database Migration Service (DMS) with Salesforce as the source and S3 as the target.

**B)** Amazon AppFlow with a Salesforce connector, configured to run hourly, outputting to S3 in Parquet format.

**C)** AWS Glue with a custom connector to the Salesforce REST API, running on a scheduled trigger every hour.

**D)** Amazon Kinesis Firehose with a Salesforce custom HTTP producer pushing records every hour.

**✅ Correct Answer: B**

**Explanation:** Amazon AppFlow is the purpose-built no-code/low-code service for SaaS integrations. It has native Salesforce connectors, supports hourly scheduling, and can output to S3 in Parquet format directly — with zero code. **A** DMS does not support Salesforce as a source (it's for databases). **C** Glue requires writing a custom connector — that's coding, which the team explicitly cannot do. **D** Firehose doesn't support Salesforce natively; Salesforce would need to push data, not pull it.

---

### Q26 — Kinesis Firehose Inline Format Conversion
**Domain 1 | Key Services: Kinesis Firehose, Glue Data Catalog**

A data team receives clickstream events via Kinesis Data Streams in JSON format. They want to deliver the data to S3 in Parquet format so Athena queries are cheaper — without writing any ETL code or deploying additional compute. The events have a consistent, known schema registered in the Glue Data Catalog. Which approach achieves inline format conversion with the least operational overhead?

**A)** Use Kinesis Firehose as a consumer of the Kinesis Data Stream. Enable the Firehose record format conversion feature, specifying the Glue Data Catalog table that defines the schema. Firehose converts JSON to Parquet during delivery with no additional compute.

**B)** Use Kinesis Firehose to deliver raw JSON to S3, then trigger a Glue ETL job via EventBridge on each file arrival to convert to Parquet.

**C)** Use Managed Service for Apache Flink to read from the stream, apply a format transformation, and write Parquet to S3.

**D)** Configure Kinesis Data Streams to output directly to S3 in Parquet format using the native format conversion setting.

**✅ Correct Answer: A**

**Explanation:** Kinesis Firehose has a built-in record format conversion feature that converts JSON to Parquet or ORC during delivery — zero additional compute or ETL jobs. The only prerequisite is a Glue Data Catalog table defining the schema (Firehose uses it for serialization). This is the purpose-built pattern for this use case. **B** adds a Glue job invocation per file — more latency, more cost, more components to fail. **C** Flink works but adds a continuously running application — far more operational overhead. **D** Kinesis Data Streams is a streaming buffer only; it delivers raw records as written and has no format conversion capability.

---

### Q27 — MSK vs Kinesis for Kafka Migration
**Domain 1 | Key Services: MSK, Kinesis**

A company is migrating a real-time fraud detection system from on-premises to AWS. The existing system uses Apache Kafka with 15 topics, consumer groups with manually managed offsets, Kafka Streams for stateful aggregations, and Kafka Connect for CDC from a PostgreSQL database. The migration must preserve all existing consumer group semantics and Kafka protocol compatibility. The team has deep Kafka expertise. Which service should they choose?

**A)** Amazon Kinesis Data Streams with Enhanced Fan-Out for multiple consumers. Rewrite Kafka Streams logic using Managed Service for Apache Flink.

**B)** Amazon MSK (Managed Streaming for Apache Kafka), which is fully Kafka-compatible — existing Kafka clients, consumer groups, Kafka Streams, and Kafka Connect work without code changes.

**C)** Amazon Kinesis Data Streams with the Kinesis Consumer Library (KCL) replicating Kafka consumer group semantics.

**D)** Amazon EventBridge with custom event buses per topic and EventBridge Pipes for stateful processing.

**✅ Correct Answer: B**

**Explanation:** MSK is managed Apache Kafka — the existing clients, consumer group protocols, Kafka Streams API, and Kafka Connect connectors all work without modification. When a scenario explicitly describes an existing Kafka deployment with Kafka-native features (consumer groups, Kafka Streams, Kafka Connect), MSK is always the answer. **A** Kinesis has fundamentally different semantics (shards vs partitions, different SDK, different retention model) — migrating would require rewriting all consumer code. **C** KCL approximates consumer group behavior but is a different API — existing Kafka client code cannot use it. **D** EventBridge cannot replicate Kafka semantics or stateful stream processing.

---

### Q28 — Lambda Parallelization Factor for Kinesis
**Domain 1 | Key Services: Lambda, Kinesis**

A Lambda function consumes records from a Kinesis Data Stream with 10 shards. `IteratorAge` is consistently high (> 5 minutes), indicating the consumer falls behind. Lambda processes each batch in approximately 8 seconds. The team has confirmed the function code is already optimized. They want to increase processing throughput **without changing the shard count or enabling Enhanced Fan-Out**. Which configuration achieves this?

**A)** Increase the Lambda batch size from 100 to 10,000 records per invocation.

**B)** Set the Lambda event source mapping `ParallelizationFactor` to 10. This allows up to 10 concurrent Lambda invocations per shard — 100 concurrent invocations across 10 shards — each processing a sequential subset of the shard.

**C)** Increase the Lambda reserved concurrency limit from 10 to 100.

**D)** Increase the Lambda timeout from 30 seconds to 5 minutes to allow larger batches to complete.

**✅ Correct Answer: B**

**Explanation:** `ParallelizationFactor` (range 1–10) enables multiple concurrent Lambda invocations per shard, each consuming a non-overlapping ordered subset of the shard's records. With 10 shards and factor=10, up to 100 concurrent Lambda invocations process records simultaneously — 10× the throughput with no shard changes. **A** a larger batch size reduces the number of invocations but each invocation still processes sequentially; total throughput doesn't increase proportionally because the function still runs one invocation per shard at a time by default. **C** reserved concurrency raises the ceiling but Kinesis only creates one concurrent invocation per shard by default — without changing `ParallelizationFactor`, the extra concurrency goes unused. **D** a longer timeout only matters if the function was timing out; it does not increase concurrency.

---

### Q29 — Glue Job Bookmarks for Incremental Processing
**Domain 1 | Key Services: Glue**

A Glue ETL job runs daily to process new log files landing in `s3://logs/year=/month=/day=/`. On day two, the team observes the job reprocessing all historical files — causing duplicate records in the output S3 location. The job currently has no incremental processing logic. What is the correct fix?

**A)** Add code to the Glue job that calls `s3.list_objects_v2` with a `LastModified` filter to read only files from the past 24 hours.

**B)** Enable Glue Job Bookmarks by setting `--job-bookmark-option job-bookmark-enable` in the job parameters and calling `job.commit()` at the end of successful processing. Glue will track which S3 keys were already processed and skip them on the next run.

**C)** Use a Glue trigger with `StartingPosition=AFTER_SEQUENCE_NUMBER` to consume only new files since the last trigger.

**D)** Add a pushdown predicate `year='2024' and month='01' and day='15'` hardcoded to today's date in the job script.

**✅ Correct Answer: B**

**Explanation:** Glue Job Bookmarks are the built-in mechanism for incremental S3 processing. When enabled, Glue records the S3 keys (and modification times) it processed at each `job.commit()`. On the next run, it skips everything before the bookmark. If a job fails before `commit()`, the bookmark is not advanced — the next run safely reprocesses the failed run's data. **A** `last_modified` filtering is fragile — S3 metadata changes on copy/replication and doesn't track Glue's logical processing position. **C** `StartingPosition=AFTER_SEQUENCE_NUMBER` is a Kinesis concept; it has no meaning in Glue S3 jobs. **D** hardcoding today's date makes the job non-rerunnable and fails on weekends or if the job is skipped for a day.

---

### Q30 — Glue DynamicFrame resolveChoice for Schema Conflicts
**Domain 1 | Key Services: Glue, DynamicFrame**

A Glue ETL job reads 3 years of event files from S3. Due to schema evolution, the `event_value` column is stored as `int` in 2021 files, `double` in 2022 files, and `string` in 2023 files. When the job calls `.toDF()` to convert the DynamicFrame to a Spark DataFrame for processing, it fails with a schema conflict error. What is the correct resolution?

**A)** Filter the input to read only 2023 files (which have `string` type) and cast downstream in Spark SQL.

**B)** Call `resolve_choice(specs=[("event_value", "cast:double")])` on the DynamicFrame before converting to a DataFrame. This casts all type variants to `double`, resolving the ambiguity before the DataFrame conversion.

**C)** Enable the `--enable-update-catalog` Glue job parameter so Glue auto-selects the most common column type.

**D)** Create separate Glue jobs for each year's files, each with a hardcoded schema, and union the results in a final Glue job.

**✅ Correct Answer: B**

**Explanation:** `DynamicFrame.resolve_choice()` is the purpose-built API for type conflicts across files with different schemas. When Glue reads files with ambiguous types, it represents them as a `choice` type in the DynamicFrame. Calling `resolve_choice(specs=[("event_value", "cast:double")])` tells Glue to coerce all variants (`int`, `double`, `string`) to `double`, producing a clean, consistent schema before `.toDF()`. **A** ignoring 2 years of data defeats the purpose of the analytics job. **C** `--enable-update-catalog` updates the Glue catalog schema when new columns are discovered — it does not resolve type conflicts for an existing column. **D** three separate jobs with a union adds complexity and doesn't leverage DynamicFrame's built-in capability.

---

### Q31 — Kinesis On-Demand vs Provisioned Mode
**Domain 1 | Key Services: Kinesis Data Streams**

A news platform has Kinesis Data Streams in Provisioned mode with 50 shards, sized for peak breaking-news traffic (50,000 events/second). On normal days, throughput drops to 500 events/second — leaving 49 shards idle. The team wants to eliminate the wasted idle shard cost without risking throttling during unpredictable traffic spikes. Operational simplicity is also required. Which change is most appropriate?

**A)** Keep Provisioned mode but add an Application Auto Scaling policy on the Kinesis stream to add/remove shards based on the `IncomingBytes` CloudWatch metric.

**B)** Switch to Kinesis Data Streams On-Demand mode. The stream automatically scales to match actual throughput (up to 200 MB/s write / 400 MB/s read by default) and you pay per GB processed — no shard management required.

**C)** Reduce provisioned shards to 5 and configure CloudWatch alarms to alert the team to manually reshard before anticipated breaking news events.

**D)** Replace Kinesis Data Streams with Kinesis Firehose, which auto-scales and has no shard concept.

**✅ Correct Answer: B**

**Explanation:** On-Demand mode is purpose-built for unpredictable, variable traffic. It automatically scales capacity for sudden spikes without any manual intervention or scaling policy, and charges per GB of data processed — not per shard-hour — eliminating idle cost. **A** Application Auto Scaling for Kinesis exists but scaling latency (adding shards takes several minutes) makes it risky for sudden breaking-news spikes. **C** manual resharding before events is operationally unreliable — breaking news is unplanned by definition. **D** Kinesis Firehose is a delivery service, not a real-time streaming buffer; it has mandatory buffering delays and cannot serve real-time consumers (Lambda, Flink, KCL applications) the way Data Streams does.

---

## Domain 2: Data Store Management

---

### Q10 — Redshift Distribution Key Selection
**Domain 2 | Key Services: Redshift**

A retail data warehouse team is designing their Redshift schema. The `fact_orders` table has 2 billion rows. The `dim_products` table has 500,000 rows. The `dim_customers` table has 10 million rows. The most common query joins `fact_orders` and `dim_customers` on `customer_id`, and frequently filters by `order_date`. Which distribution and sort key configuration is most appropriate for `fact_orders`?

**A)** DISTKEY(order_date) SORTKEY(customer_id)

**B)** DISTSTYLE EVEN, SORTKEY(order_date)

**C)** DISTKEY(customer_id) SORTKEY(order_date), and DISTKEY(customer_id) on `dim_customers`

**D)** DISTSTYLE ALL, SORTKEY(order_date)

**✅ Correct Answer: C**

**Explanation:** The most frequent join is on `customer_id` — using KEY distribution on both `fact_orders` and `dim_customers` with the same key ensures co-located joins (no data movement). The most frequent filter is `order_date` — compound sort key on `order_date` enables zone map skipping. `dim_products` at 500K rows → ALL distribution is appropriate (small table, copied to every node). **A** distributing on `order_date` for the fact table makes the join on `customer_id` expensive. **B** EVEN with sort on `order_date` is fine for filtering but the join on `customer_id` will require shuffle. **D** ALL distribution on a 2-billion-row fact table is catastrophic — it would replicate 2B rows to every node.

---

### Q11 — DynamoDB RCU Calculation
**Domain 2 | Key Services: DynamoDB**

A product catalog application reads product details for an e-commerce website. Each product item is 6 KB. The application performs 200 reads per second, and the business has accepted eventual consistency for the product catalog (prices update in the background, slight delay acceptable). How many RCUs should be provisioned?

**A)** 200 RCUs

**B)** 300 RCUs

**C)** 100 RCUs

**D)** 600 RCUs

**✅ Correct Answer: C**

**Explanation:**
- Item size: 6 KB
- RCU for strong consistency: ceil(6 KB / 4 KB) = 2 RCU per read
- RCU for **eventual consistency**: 2 / 2 = **1 RCU per read**
- Total: 200 reads/sec × 1 RCU = **100 RCUs**

**A** (200) would be correct for strong consistency. **B** (300) has no basis. **D** (600) would be correct for transactional reads at strong consistency. The key insight: eventual consistency halves the RCU cost.

---

### Q12 — Open Table Format Selection
**Domain 2 | Key Services: Iceberg, Hudi, Athena**

A healthcare analytics company stores 10 years of patient records in S3 as Parquet files. They need to: (1) comply with HIPAA's data rectification requirements by updating incorrect records, (2) delete records for patients who opt out, (3) query historical data as it existed on a specific date for audit purposes, and (4) query the data using Athena without loading it into a database. Which solution best meets all four requirements?

**A)** Store data in Delta Lake format on S3 and query with Amazon EMR Spark.

**B)** Store data in Apache Iceberg format on S3. Use AWS Glue for DML operations (UPDATE, DELETE, MERGE) and Amazon Athena for queries.

**C)** Store data in Apache Hudi with Merge-on-Read tables on S3 and query with EMR.

**D)** Use Amazon DynamoDB for record storage and export to S3 periodically for Athena queries.

**✅ Correct Answer: B**

**Explanation:** Apache Iceberg meets all four requirements: (1) UPDATE via MERGE INTO, (2) DELETE by row, (3) time travel to query data as of a specific timestamp, (4) native Athena support (Athena supports Iceberg natively — no EMR required). **A** Delta Lake lacks native Athena support — you'd need EMR, which adds complexity. **C** Hudi works but native Athena support for Hudi is more limited than Iceberg; MOR adds read complexity. **D** DynamoDB doesn't support SQL time travel and the periodic export creates staleness — fails requirement 3.

---

### Q13 — S3 Table Format and Compaction
**Domain 2 | Key Services: S3 Tables, Iceberg**

A startup is building a new data lakehouse from scratch. They expect high-frequency writes (streaming from Kinesis) creating many small files, and they want Iceberg ACID capabilities. Their data engineering team is small (2 people) and cannot spend time managing compaction jobs, snapshot cleanup, or metadata maintenance. Which solution minimizes operational overhead?

**A)** Apache Iceberg tables on standard S3, with a weekly Glue job running `rewrite_data_files` for compaction and a Lambda function running `expire_snapshots` on a schedule.

**B)** Amazon S3 Tables, which provides managed Iceberg tables with automatic compaction, snapshot expiry, and metadata management built in.

**C)** Apache Hudi with Copy-on-Write tables, which eliminates the need for separate compaction jobs.

**D)** Delta Lake with OPTIMIZE and VACUUM configured as scheduled Glue jobs.

**✅ Correct Answer: B**

**Explanation:** S3 Tables (Amazon's managed Iceberg offering) is specifically designed for the "Iceberg with zero operational overhead" use case. AWS handles compaction, snapshot management, and metadata maintenance automatically. **A** works but requires managing and monitoring two separate maintenance jobs — exactly the overhead the team wants to avoid. **C** Hudi CoW still requires periodic compaction; writes just do it inline per write rather than batch — doesn't eliminate compaction, just changes when it happens. **D** Delta Lake requires OPTIMIZE and VACUUM jobs — more maintenance, not less.

---

### Q14 — Lake Formation vs S3 Policies
**Domain 2 | Key Services: Lake Formation, Athena**

A data analyst has full S3 read access via an IAM policy (`s3:GetObject` on `arn:aws:s3:::data-lake-bucket/*`) and Athena permissions. However, when they run an Athena query on the `orders` table, they receive AccessDenied. The table metadata is in Glue Data Catalog, and the S3 location is registered with Lake Formation. What is the most likely cause?

**A)** The analyst's IAM policy is missing `athena:GetQueryResults`.

**B)** The analyst's IAM role doesn't have `glue:GetTable` permission.

**C)** Lake Formation has not granted the analyst SELECT permission on the `orders` table. Despite having S3 IAM access, Lake Formation controls data access for registered locations.

**D)** The S3 bucket policy doesn't allow the Athena service principal.

**✅ Correct Answer: C**

**Explanation:** When an S3 location is registered with Lake Formation, Lake Formation permissions **supersede** S3 bucket policies for query engines that honor Lake Formation (Athena, Redshift Spectrum, EMR). Even with full S3 IAM read access, the analyst needs a Lake Formation SELECT grant on the `orders` table. This is the most common misconfiguration when onboarding users to a governed data lake. **A** and **B** might cause different errors but not the specific scenario where Athena queries fail on Lake Formation-governed tables. **D** would cause a different type of error and isn't the typical pattern.

---

### Q15 — Redshift Sort Key vs WLM
**Domain 2 | Key Services: Redshift**

A Redshift data warehouse serves both a BI reporting team (running 5-10 complex aggregation queries per minute on the previous day's data, filtered by `report_date`) and a data engineering team (running nightly batch loads via COPY commands and 2-3 hour SQL transformations). BI users complain that their dashboard queries sometimes take 5+ minutes instead of the usual 30 seconds. The ETL jobs run between 2 AM and 6 AM. What is the most effective combination of changes?

**A)** Add a compound sort key on `report_date` to all reporting tables. Enable Auto WLM with Short Query Acceleration (SQA).

**B)** Switch to an interleaved sort key with multiple columns. Increase cluster size to add more nodes.

**C)** Schedule ETL jobs to run during business hours to avoid the issue. Add compound sort key on report_date.

**D)** Enable Concurrency Scaling. Move all ETL to EMR Spark to reduce cluster load.

**✅ Correct Answer: A**

**Explanation:** Two issues: (1) The sort key on `report_date` enables zone map skipping — if data is sorted by date, Redshift skips entire blocks for other dates, making yesterday's data queries much faster. (2) Auto WLM + SQA automatically routes fast queries (the 30-second BI queries) to a fast lane, bypassing the queue behind slower queries. These together address both the query performance and the queuing problem. **B** Interleaved sort key is not the right choice when there's a primary filter column (`report_date`); compound is better. Cluster resizing is expensive and doesn't fix the root cause. **C** Running ETL during business hours would directly compete with BI queries — worse situation. **D** Concurrency Scaling helps with burst but moving ETL to EMR doesn't fix the BI query performance problem and adds complexity.

---

## Domain 3: Data Operations and Support

---

### Q16 — Data Quality Gates
**Domain 3 | Key Services: Glue Data Quality, Step Functions**

A financial services company runs a daily ETL pipeline that loads transaction data from S3 into Amazon Redshift. The pipeline has no data quality checks. Last month, a corrupt source file (with null transaction IDs) was loaded into Redshift, causing downstream risk reports to produce incorrect values. The compliance team requires that: (1) any batch with null transaction IDs must not be loaded, (2) batches with <10,000 records must be flagged as suspicious and held for review, (3) all data quality results must be auditable. Which solution meets all three requirements?

**A)** Add a Lambda function before the Redshift COPY command that counts rows and checks for nulls. If checks fail, skip the load and log to CloudWatch.

**B)** Use AWS Glue Data Quality with DQDL rules: `IsPrimaryKey "transaction_id"` with FAIL action, and `RowCount > 10000` with QUARANTINE action. Wrap in Step Functions. Write DQ results to CloudWatch Logs for audit.

**C)** Use Amazon DataBrew to profile the data before each load. If profiling detects issues, send a notification to the compliance team via SNS.

**D)** Add a `WHERE transaction_id IS NOT NULL` filter to the COPY command and configure Redshift to reject rows with errors (MAXERROR 0).

**✅ Correct Answer: B**

**Explanation:** Glue Data Quality with DQDL directly addresses all three requirements: (1) `IsPrimaryKey` rule with FAIL action prevents any null transaction IDs from loading, (2) `RowCount > 10000` with QUARANTINE action flags suspicious batches while still processing valid rows (or can be set to FAIL to hold entire batch), (3) DQ results written to CloudWatch Logs provide an auditable, timestamped trail. Step Functions wraps the workflow with proper error handling. **A** Lambda could work but is custom code that's harder to maintain and audit. **C** DataBrew profiling is manual and slow — not suitable for daily automated pipelines. **D** The COPY filter removes the bad rows but doesn't prevent the load of a batch that's too small, and provides no audit trail.

---

### Q17 — Kinesis Error Handling
**Domain 3 | Key Services: Kinesis, Lambda, SQS**

A Lambda function processes records from a Kinesis Data Stream in batches of 100. One record in a batch causes the Lambda function to throw an unhandled exception. The function fails and Lambda retries the entire batch. The same bad record continues to cause failures, blocking all records behind it in the shard. The team needs to: isolate bad records without blocking the shard, preserve failed records for investigation, and limit retry attempts. Which configuration resolves this?

**A)** Reduce the batch size to 1 so each record is processed independently. If Lambda fails, only that record blocks.

**B)** Set `bisectBatchOnFunctionError: true`, `maximumRetryAttempts: 3`, `maximumRecordAgeInSeconds: 3600`, and configure `destinationConfig.onFailure` pointing to an SQS DLQ.

**C)** Switch from Kinesis to SQS FIFO, which handles failed messages by sending them to a DLQ automatically.

**D)** Increase Lambda timeout to 15 minutes and add try/except around each record in the batch so the function always returns success.

**✅ Correct Answer: B**

**Explanation:** `bisectBatchOnFunctionError: true` splits the failing batch in half on each failure, quickly isolating the bad record in O(log n) retries. `maximumRetryAttempts: 3` prevents infinite retries. `maximumRecordAgeInSeconds: 3600` ensures that even if the bad record persists, it eventually gets skipped after 1 hour. The SQS DLQ receives the failed records for investigation. **A** batch size of 1 eliminates efficiency and doesn't prevent the single bad record from blocking the shard indefinitely. **C** switching to SQS changes the entire architecture and loses Kinesis replay capability. **D** swallowing exceptions hides failures — the bad records are silently dropped with no investigation path.

---

### Q18 — CloudWatch Monitoring Strategy
**Domain 3 | Key Services: CloudWatch, Glue, Kinesis**

A data platform team maintains 50 Glue ETL jobs, 3 Kinesis streams, and 2 EMR clusters. They want to detect: (1) Glue jobs that take >2x their baseline duration, (2) Kinesis consumers falling behind, (3) EMR cluster running with high YARN memory utilization. What is the most scalable monitoring setup?

**A)** Write a Lambda function that queries the AWS API every 5 minutes, checks each service's metrics, and sends email if thresholds are exceeded.

**B)** Use CloudWatch Metric Alarms: alarm on `glue.driver.aggregate.elapsedTime` per job, alarm on `IteratorAgeMilliseconds` per Kinesis stream, and alarm on `YARNMemoryAvailablePercentage` per EMR cluster. Route all alarms to SNS → Slack webhook.

**C)** Use AWS Config rules to detect when jobs exceed their expected duration.

**D)** Enable CloudWatch Container Insights for EMR and use X-Ray for Glue job tracing.

**✅ Correct Answer: B**

**Explanation:** CloudWatch Metric Alarms are the native, scalable solution for each requirement. Each metric maps directly: Glue job duration via `glue.driver.aggregate.elapsedTime`, Kinesis consumer lag via `IteratorAgeMilliseconds`, EMR YARN utilization via `YARNMemoryAvailablePercentage`. SNS routes to Slack for team notifications. **A** custom Lambda polling is an anti-pattern — CloudWatch alarms already do this natively, and polling every 5 minutes adds latency. **C** AWS Config is for resource configuration compliance (e.g., "are S3 buckets encrypted?"), not operational metrics. **D** Container Insights and X-Ray are useful for different purposes but don't directly solve all three monitoring requirements.

---

### Q19 — Cost Optimization in Data Pipelines
**Domain 3 | Key Services: Athena, S3, Glue**

A startup runs ad-hoc analytics on 50 TB of event data stored in S3 as uncompressed JSON files. Their Athena costs are $3,000/month despite running only 100 queries/day. Each query scans the entire dataset. The data is partitioned by `event_date`. What changes would most significantly reduce Athena query costs?

**A)** Convert data to Parquet format with Snappy compression using a Glue ETL job. Add `event_date` partition pruning to all queries using WHERE clauses. Enable Athena query result caching.

**B)** Move data to Redshift Serverless and use Redshift SQL instead of Athena.

**C)** Use Amazon EMR Spark for all queries instead of Athena.

**D)** Enable S3 Intelligent-Tiering on the data. This reduces storage costs, which reduces Athena scan costs.

**✅ Correct Answer: A**

**Explanation:** Athena charges per TB scanned. Three changes dramatically reduce scan volume: (1) Parquet is columnar — if queries select 5 of 50 columns, Athena reads only those 5 columns. Snappy compression reduces file size 3-5×. Combined: potential 10-20× cost reduction. (2) Partition pruning prevents full dataset scans — queries with `WHERE event_date = '2024-01-15'` scan only that day's data, not all 50 TB. (3) Query result caching serves repeated identical queries from cache with no scan cost. **B** Redshift Serverless would work but is more expensive for ad-hoc infrequent queries and requires data loading. **C** EMR adds cluster management overhead. **D** S3 storage pricing doesn't affect Athena per-TB-scanned charges — these are separate cost dimensions.

---

### Q20 — Step Functions Error Handling
**Domain 3 | Key Services: Step Functions**

A data pipeline uses Step Functions Standard Workflow to orchestrate a Glue ETL job, a data quality check, and a Redshift COPY command. The Glue job occasionally fails due to transient AWS service issues (`Lambda.ServiceException`). The team wants: up to 3 retries with exponential backoff for transient errors, but if the data quality check fails with a custom error `DataQualityFailed`, the workflow should immediately go to an error notification state without retrying, and the original input data should be preserved in the notification state.

Which Retry/Catch configuration is correct for the data quality state?

**A)**
```json
"Retry": [{"ErrorEquals": ["States.ALL"], "MaxAttempts": 3}],
"Catch": [{"ErrorEquals": ["DataQualityFailed"], "Next": "NotifyError", "ResultPath": "$"}]
```

**B)**
```json
"Retry": [{"ErrorEquals": ["DataQualityFailed"], "MaxAttempts": 0}],
"Catch": [{"ErrorEquals": ["DataQualityFailed"], "Next": "NotifyError", "ResultPath": "$.error"}]
```

**C)**
```json
"Retry": [
  {"ErrorEquals": ["Lambda.ServiceException"], "IntervalSeconds": 2, "MaxAttempts": 3, "BackoffRate": 2.0},
  {"ErrorEquals": ["DataQualityFailed"], "MaxAttempts": 0}
],
"Catch": [{"ErrorEquals": ["States.ALL"], "Next": "NotifyError", "ResultPath": "$.error"}]
```

**D)**
```json
"Retry": [{"ErrorEquals": ["Lambda.ServiceException"], "MaxAttempts": 3}],
"Catch": [{"ErrorEquals": ["States.ALL"], "Next": "NotifyError", "ResultPath": "$.error"},
          {"ErrorEquals": ["DataQualityFailed"], "Next": "NotifyError", "ResultPath": "$"}]
```

**✅ Correct Answer: C**

**Explanation:** Option C correctly: (1) Retries `Lambda.ServiceException` up to 3 times with exponential backoff (IntervalSeconds + BackoffRate), (2) Sets `MaxAttempts: 0` for `DataQualityFailed` which means zero retries — immediately falls through to Catch, (3) Uses `ResultPath: "$.error"` which **adds** the error to the original input rather than replacing it (`"$"` would overwrite). The Catch on `States.ALL` catches any error (including `DataQualityFailed` after 0 retries) and routes to `NotifyError`. **A** `ResultPath: "$"` overwrites original input — preserves nothing. **B** doesn't configure Lambda service retries. **D** has two Catch entries for overlapping patterns — the first (`States.ALL`) would catch everything before the second entry is evaluated.

---

## Domain 4: Data Security and Governance

---

### Q21 — Lake Formation Column Security
**Domain 4 | Key Services: Lake Formation, Athena**

A healthcare analytics company stores patient records in S3 with the Glue Data Catalog managing the schema. The `patients` table has columns: `patient_id`, `name`, `dob`, `ssn`, `diagnosis`, `treatment_code`. Data analysts should be able to query diagnoses and treatment codes for population health studies but must never see `ssn`, `name`, or `dob`. Compliance requires that this restriction is enforced at the data access layer, not just in application code. Which solution is most appropriate?

**A)** Create an Athena view that excludes `ssn`, `name`, and `dob` columns and grant analysts access to the view only.

**B)** Use AWS Lake Formation column-level security. Grant analysts SELECT permission on the `patients` table with `ssn`, `name`, and `dob` excluded via the column wildcard with exclusions.

**C)** Encrypt the `ssn`, `name`, and `dob` columns with a KMS key that analysts don't have access to.

**D)** Create separate S3 prefixes for PII and non-PII columns and grant S3 read access only to the non-PII prefix.

**✅ Correct Answer: B**

**Explanation:** Lake Formation column-level security is the authoritative data-layer control. It's enforced by Athena, Glue, and EMR — not just one application. Even if an analyst bypasses the application and queries directly via Athena console, they still cannot see the excluded columns. **A** views can be bypassed if analysts have direct table access. Also, Athena views don't prevent users who have direct table grants from querying the full table. **C** encryption at the column level is not a native S3/Glue feature; KMS encrypts entire files, not individual columns. **D** separating into different S3 paths requires completely restructuring the data model and breaks the atomicity of a patient record.

---

### Q22 — IAM Policy Evaluation
**Domain 4 | Key Services: IAM, S3, SCP**

A developer in an AWS account is a member of the `DataEngineers` IAM group, which has a policy allowing `s3:*` on all S3 resources. The developer also has an inline policy attached directly to their user that explicitly denies `s3:DeleteObject`. The company's AWS Organization SCP allows all S3 actions for this account. The developer tries to delete an S3 object. What happens?

**A)** The delete succeeds because the group policy's `s3:*` allows all S3 actions, which overrides the user's inline deny.

**B)** The delete is denied because the SCP doesn't explicitly allow `s3:DeleteObject` at the action level.

**C)** The delete is denied because the explicit deny in the user's inline policy overrides the group Allow — explicit Deny wins in all cases.

**D)** The delete succeeds because the SCP allows all S3 actions, which takes precedence over individual account policies.

**✅ Correct Answer: C**

**Explanation:** **Explicit Deny always wins** — this is the cardinal IAM rule. An explicit `Deny` on `s3:DeleteObject` in the user's inline policy overrides the `Allow` from the group policy, regardless of the scope or breadth of the Allow. There is no way to override an explicit Deny except by removing the Deny statement itself. **A** is wrong — explicit Deny is never overridden by Allow. **B** is wrong — the SCP allows all S3 actions in this scenario. **D** is wrong — SCPs restrict permissions, they don't grant permissions that override local policies.

---

### Q23 — KMS and Encryption
**Domain 4 | Key Services: KMS, S3, CloudTrail**

A financial institution is implementing a compliance requirement that mandates: (1) all data in S3 must be encrypted, (2) encryption keys must be rotatable annually, (3) every access to an encryption key must be audited with the identity of the requester, the action performed, and the timestamp. Which encryption option satisfies all three requirements?

**A)** SSE-S3 (Server-Side Encryption with S3-managed keys, AES-256)

**B)** SSE-KMS with an AWS-managed key (aws/s3)

**C)** SSE-KMS with a Customer Managed Key (CMK) and CloudTrail enabled

**D)** Client-side encryption with keys stored in your on-premises HSM

**✅ Correct Answer: C**

**Explanation:** SSE-KMS with a CMK satisfies all three: (1) data encrypted at rest in S3, (2) Customer Managed Keys support annual rotation (`EnableKeyRotation: true`), (3) every `GenerateDataKey` and `Decrypt` KMS API call is logged in CloudTrail with requester identity, timestamp, and action. **A** SSE-S3 uses AWS-managed keys with no customer visibility — no audit trail of key usage, no customer rotation control. **B** AWS-managed keys (aws/s3) satisfy encryption and CloudTrail logging, but AWS-managed keys rotate every 3 years automatically — customer cannot control rotation schedule. Also, AWS-managed keys have less granular access policy control. **D** client-side encryption satisfies encryption but requires complex key management infrastructure and doesn't integrate with CloudTrail for AWS-side auditing.

---

### Q24 — VPC Endpoint and Data Perimeter
**Domain 4 | Key Services: VPC Endpoints, S3**

A large enterprise wants to prevent data exfiltration from their AWS VPC. Specifically, they want to ensure that EC2 instances and Glue jobs running within their VPC **cannot upload data to any S3 bucket outside their AWS Organization**, even if an employee has valid AWS credentials and IAM permissions to external buckets. Which mechanism enforces this restriction at the network level?

**A)** Add an IAM policy deny for `s3:PutObject` on buckets with ARNs not matching the company's organization pattern.

**B)** Configure a VPC Endpoint Policy on the S3 Gateway Endpoint that uses the condition `aws:ResourceOrgID` to restrict access to buckets owned by the company's AWS Organization.

**C)** Use Amazon Macie to scan for any data uploaded to external buckets and alert security teams.

**D)** Configure S3 bucket policies on all internal buckets to deny access from outside the VPC.

**✅ Correct Answer: B**

**Explanation:** The VPC Endpoint Policy on the S3 Gateway Endpoint is a network-level control. ALL S3 traffic from resources in the VPC flows through the Gateway Endpoint (since it's in the route table). The endpoint policy with `aws:ResourceOrgID` condition allows S3 requests only to buckets belonging to the company's AWS Organization — requests to external buckets are denied at the endpoint, before they even reach S3. **A** IAM policies are identity-based — an employee with access to a personal account could temporarily escalate permissions or use credentials from another source. **C** Macie is detective (finds violations after they happen), not preventive. **D** Internal bucket policies prevent external access TO your data, not exfiltration FROM your VPC to external buckets.

---

### Q25 — Permission Boundaries and Privilege Escalation
**Domain 4 | Key Services: IAM, Permission Boundaries**

A company's DevOps team has `AdministratorAccess` to deploy and manage infrastructure. Security is concerned that developers could use this access to create IAM roles with `AdministratorAccess` for their applications, effectively giving their apps unlimited AWS permissions. The security team wants developers to be able to create IAM roles for their applications but restrict those roles to only the S3 and DynamoDB permissions their apps actually need. Which mechanism achieves this without removing developers' ability to manage infrastructure?

**A)** Create a new IAM group `AppDevelopers` with only `s3:*` and `dynamodb:*` permissions. Move all developers to this group.

**B)** Attach a Service Control Policy (SCP) to the developers' AWS account that denies IAM role creation.

**C)** Create a Permission Boundary policy that allows only `s3:*` and `dynamodb:*`. Require developers to attach this boundary to any IAM roles they create using a conditional IAM policy: allow `iam:CreateRole` only if `iam:PermissionsBoundary` equals the approved boundary ARN.

**D)** Enable AWS Organizations and move developer accounts to a separate OU with restricted permissions.

**✅ Correct Answer: C**

**Explanation:** Permission Boundaries are specifically designed to prevent privilege escalation in delegated administration scenarios. The permission boundary limits the *maximum* permissions of any role a developer creates — even if they attach `AdministratorAccess` to the role, the boundary caps it at `s3:*` and `dynamodb:*`. The conditional IAM policy ensures developers must attach the boundary when creating roles. **A** restricts the developers themselves, but doesn't solve the problem — if they somehow still have IAM permissions, they can create roles without the boundary. **B** SCPs prevent role creation entirely — over-restrictive, prevents legitimate role creation for apps. **D** OU restructuring doesn't solve the within-account privilege escalation problem.

---

## Quick Reference — Answer Key

| Q | Answer | Domain | Key Concept |
|---|--------|--------|-------------|
| 1 | B | D1 | Enhanced Fan-Out for multiple consumers |
| 2 | D | D1 | Optimize before migrating platforms |
| 3 | C | D1 | Shard math — records/sec is binding constraint |
| 4 | B | D1 | DMS Full Load + CDC for zero-downtime migration |
| 5 | B | D1 | Existing Airflow + backfill → MWAA |
| 6 | B | D1 | Glue Streaming checkpoint location |
| 7 | B | D1 | Session windows → Flink, not Lambda |
| 8 | C | D1 | Variable workload + no DevOps → EMR Serverless |
| 9 | B | D1 | SaaS no-code integration → AppFlow |
| 26 | A | D1 | Firehose inline JSON→Parquet conversion via Glue Catalog schema |
| 27 | B | D1 | Kafka migration with consumer groups + Kafka Streams → MSK |
| 28 | B | D1 | Lambda ParallelizationFactor increases throughput per shard |
| 29 | B | D1 | Glue Job Bookmarks for incremental S3 processing |
| 30 | B | D1 | DynamicFrame resolveChoice for cross-file type conflicts |
| 31 | B | D1 | Kinesis On-Demand eliminates idle shard cost for variable traffic |
| 10 | C | D2 | Distribution KEY on join column, SORT on filter column |
| 11 | C | D2 | Eventual consistency = half the RCU cost |
| 12 | B | D2 | Iceberg for ACID + time travel + native Athena |
| 13 | B | D2 | Zero operational overhead → S3 Tables |
| 14 | C | D2 | Lake Formation supersedes S3 IAM for governed tables |
| 15 | A | D2 | Sort key + Auto WLM + SQA for mixed workloads |
| 16 | B | D3 | Glue DQ DQDL with FAIL + QUARANTINE actions |
| 17 | B | D3 | bisectBatchOnFunctionError + maxRetries + DLQ |
| 18 | B | D3 | Native CloudWatch alarms per service metric |
| 19 | A | D3 | Parquet + partition pruning + caching → reduce scan |
| 20 | C | D3 | Step Functions Retry order + ResultPath: "$.error" |
| 21 | B | D4 | Lake Formation column exclusion at data layer |
| 22 | C | D4 | Explicit Deny always wins |
| 23 | C | D4 | CMK = rotation control + CloudTrail audit |
| 24 | B | D4 | VPC Endpoint Policy + aws:ResourceOrgID |
| 25 | C | D4 | Permission Boundaries for delegated role creation |
