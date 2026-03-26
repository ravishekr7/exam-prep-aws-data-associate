# Programming Concepts and Infrastructure as Code

Covers programming, IaC, CI/CD, and software engineering concepts for the exam.

---

## Programming Languages in Scope

| Language | Where Used |
|----------|-----------|
| **Python** | Glue ETL, Lambda, MWAA DAGs, EMR (PySpark) |
| **SQL** | Athena, Redshift, Glue (SparkSQL), RDS |
| **Scala** | Glue ETL, EMR (Spark) |
| **Java** | EMR, Kinesis KPL/KCL |
| **Bash** | EMR bootstrap actions, automation scripts |
| **R** | SageMaker, EMR |

---

## SQL for Data Engineers (Exam-Critical)

SQL is tested directly in DEA-C01 — not just "know what SQL is" but actually reading queries and identifying correct syntax for data transformation scenarios. Focus on these areas:

**Window Functions (heavily tested):**

Window functions perform calculations across a set of rows related to the current row, without collapsing them into a single output row like GROUP BY does.

```sql
-- Syntax template
function_name() OVER (
  PARTITION BY column       -- divide rows into groups (optional)
  ORDER BY column           -- order within each partition
  ROWS/RANGE BETWEEN ...    -- define the frame (optional)
)
```

| Function | What It Does | Exam Use Case |
|---|---|---|
| `ROW_NUMBER()` | Unique sequential number per partition (1,2,3...) | Deduplicate — keep only the latest record per customer |
| `RANK()` | Rank with gaps for ties (1,1,3...) | Rank products by sales, ties share rank |
| `DENSE_RANK()` | Rank without gaps for ties (1,1,2...) | Top-N with ties included |
| `LAG(col, n)` | Value of col from N rows BEFORE current row | Calculate day-over-day change in revenue |
| `LEAD(col, n)` | Value of col from N rows AFTER current row | Look ahead to next event timestamp |
| `SUM() OVER` | Running total across partition | Cumulative revenue per month |
| `COUNT() OVER` | Running count | Count events per session |

```sql
-- Deduplication: keep only the most recent record per user_id
WITH ranked AS (
  SELECT *,
    ROW_NUMBER() OVER (
      PARTITION BY user_id
      ORDER BY updated_at DESC
    ) AS rn
  FROM users
)
SELECT * FROM ranked WHERE rn = 1;

-- Running total of sales by region
SELECT
  region,
  sale_date,
  amount,
  SUM(amount) OVER (
    PARTITION BY region
    ORDER BY sale_date
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
  ) AS running_total
FROM sales;

-- Day-over-day revenue change
SELECT
  sale_date,
  revenue,
  LAG(revenue, 1) OVER (ORDER BY sale_date) AS prev_day_revenue,
  revenue - LAG(revenue, 1) OVER (ORDER BY sale_date) AS day_over_day_change
FROM daily_revenue;
```

**CTEs (Common Table Expressions):**

CTEs improve readability and allow you to break complex queries into named steps. The exam uses CTEs in scenario questions.

```sql
-- Multi-step transformation using CTEs
WITH
raw_orders AS (
  SELECT order_id, customer_id, amount, status
  FROM orders
  WHERE status != 'CANCELLED'
),
customer_totals AS (
  SELECT customer_id, SUM(amount) AS total_spent, COUNT(*) AS order_count
  FROM raw_orders
  GROUP BY customer_id
),
high_value AS (
  SELECT customer_id
  FROM customer_totals
  WHERE total_spent > 1000
)
SELECT c.customer_id, c.total_spent
FROM customer_totals c
JOIN high_value h ON c.customer_id = h.customer_id;
```

**Partition Pruning — Critical for Cost and Performance:**

Partitioned tables (S3/Athena, Redshift, Hive) only scan relevant partitions when the WHERE clause filters on the partition column. Getting this wrong wastes money and time.

```sql
-- Table partitioned by year/month/day in S3:
-- s3://bucket/events/year=2024/month=01/day=15/

-- GOOD: includes partition column in WHERE → Athena only scans Jan 2024 data
SELECT * FROM events
WHERE year = '2024' AND month = '01';

-- BAD: no partition filter → full table scan, reads ALL partitions
SELECT * FROM events
WHERE event_type = 'PURCHASE';

-- ALSO BAD: applying a function to partition column breaks pruning
SELECT * FROM events
WHERE CAST(year AS INT) = 2024;  -- function prevents partition pruning
-- Fix: WHERE year = '2024'  (match the stored type exactly)
```

Exam pattern: "Athena query is scanning more data than expected and costs are high" → check whether the WHERE clause filters on partition columns and whether functions are applied to partition columns (breaking pruning).

**UNNEST for Arrays (Athena / Presto):**

```sql
-- Flatten an array column into rows
SELECT user_id, tag
FROM users
CROSS JOIN UNNEST(tags) AS t(tag);

-- Input:  user_id=1, tags=['aws','python','spark']
-- Output: user_id=1, tag='aws'
--         user_id=1, tag='python'
--         user_id=1, tag='spark'
```

**Date/Time Functions (Athena-style, commonly tested):**

```sql
-- Truncate to month (group daily data into monthly buckets)
SELECT date_trunc('month', event_time) AS month, COUNT(*) AS events
FROM clickstream
GROUP BY date_trunc('month', event_time);

-- Difference between two timestamps in days
SELECT date_diff('day', created_at, updated_at) AS days_open
FROM tickets;

-- Current timestamp minus 7 days (freshness check)
SELECT * FROM events
WHERE event_time >= now() - interval '7' day;
```

**Exam Gotchas for SQL:**
- `GROUP BY` collapses rows — use window functions when you need both the aggregate AND the original row
- `HAVING` filters on aggregated values (after GROUP BY); `WHERE` filters before aggregation
- In Redshift, `SELECT *` reads all columns including those stored in separate micro-partitions — always select only the columns you need for columnar performance
- Athena charges per data scanned — a query with no partition filter on a 10 TB table costs significantly more than one with a partition filter that scans 1 GB

---

## ETL vs ELT — Know the Difference

This distinction appears in exam scenario questions. The correct answer depends on WHERE the transformation happens.

| Aspect | ETL (Extract → Transform → Load) | ELT (Extract → Load → Transform) |
|---|---|---|
| **Where transform happens** | Outside the destination (Glue, Spark, Lambda) | Inside the destination (Redshift SQL, Athena CTAS) |
| **Data loaded to destination** | Already cleaned and structured | Raw / semi-structured |
| **When to use** | Destination has limited compute; need to reduce data volume before loading; strict schema required at load time | Destination is powerful (Redshift, BigQuery); storage is cheap; want to preserve raw data for re-processing |
| **AWS services** | AWS Glue → S3/Redshift; EMR Spark → S3 | S3 (raw) → Athena CTAS; Redshift COPY then SQL transforms |
| **Latency** | Higher (transform is a separate step) | Lower time-to-load (raw data lands immediately) |

**ETL pattern (Glue → Redshift):**
```
S3 (raw JSON) → Glue ETL job (parse, clean, convert to Parquet) → S3 (clean Parquet) → Redshift COPY
```

**ELT pattern (load raw, transform in Redshift):**
```
S3 (raw JSON) → Redshift COPY into staging table (raw) → Redshift SQL transforms → production tables
```

**Exam patterns:**
- "Team wants to preserve raw data AND create transformed views for analysts — least operational overhead" → ELT: load raw to S3/Redshift staging, transform with SQL in place
- "Destination database has limited compute and cannot handle transformation workloads" → ETL: transform before loading
- "Pipeline needs to reduce a 1 TB dataset to 50 GB before it can fit in the destination" → ETL (reduce volume at transformation stage)
- "Redshift query performance is slow because analysts are re-running expensive transforms on every query" → materialise the transform — either ETL to pre-compute it, or use Redshift materialised views

---

## Idempotency in Data Pipelines

Idempotency is explicitly in the DEA-C01 exam guide (Task Statement 1.4). An idempotent operation produces the same result regardless of how many times it is executed. This is critical for pipeline reliability because retries, replays, and duplicate events are normal in distributed systems.

**Why it matters in data engineering:**
- Kinesis delivers records at-least-once — duplicates are possible
- Lambda retries failed async invocations up to 2 times by default
- Glue job reruns (after failure) may reprocess the same S3 files
- Step Functions Express Workflows are at-least-once — a state may execute more than once

**Common idempotency patterns:**

**1. Upsert instead of Insert:**
```sql
-- Non-idempotent: running twice creates duplicate rows
INSERT INTO orders (order_id, amount) VALUES ('123', 99.99);

-- Idempotent: second run updates the existing row instead of inserting
INSERT INTO orders (order_id, amount)
VALUES ('123', 99.99)
ON CONFLICT (order_id) DO UPDATE SET amount = EXCLUDED.amount;

-- Redshift equivalent (no ON CONFLICT — use MERGE or DELETE+INSERT)
DELETE FROM orders WHERE order_id = '123';
INSERT INTO orders (order_id, amount) VALUES ('123', 99.99);
```

**2. Deduplication with unique IDs:**
```python
# Lambda processing Kinesis records — check for duplicates using DynamoDB
import boto3
from datetime import datetime

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('processed_events')

def handler(event, context):
    for record in event['Records']:
        event_id = record['kinesis']['sequenceNumber']

        # Conditional write — only process if not already seen
        try:
            table.put_item(
                Item={'event_id': event_id, 'processed_at': str(datetime.now())},
                ConditionExpression='attribute_not_exists(event_id)'
            )
        except dynamodb.meta.client.exceptions.ConditionalCheckFailedException:
            # Already processed — skip (idempotent behaviour)
            continue

        # Process the record only if insert succeeded
        process_record(record)
```

**3. S3 partition overwrite (Glue/Spark):**
```python
# Non-idempotent: appending to same S3 path creates duplicate data on rerun
df.write.mode("append").parquet("s3://bucket/data/year=2024/month=01/")

# Idempotent: overwrite the entire partition on each run
df.write.mode("overwrite").parquet("s3://bucket/data/year=2024/month=01/")
# Second run replaces the partition entirely — no duplicates
```

**4. Glue job bookmarks for idempotency:**
- Bookmarks track which S3 objects have been processed
- On rerun: Glue only processes NEW files, skips already-processed ones
- Limitation: if the same filename is re-uploaded with new content, bookmark does NOT detect it (tracks by filename + modification time)

**Exam patterns:**
- "Lambda processes Kinesis records and inserts into DynamoDB. How to prevent duplicate rows if a record is delivered twice?" → Use DynamoDB conditional write (`attribute_not_exists`) as an idempotency check
- "Glue job fails halfway through and is rerun — how to avoid duplicate data in the output S3 path?" → Write with `mode("overwrite")` on the partition, or enable Glue job bookmarks
- "Design a pipeline that can safely be replayed from the beginning without corrupting the destination" → Use upsert/overwrite semantics at every write step

---

## Fan-Out and Fan-In Patterns

Fan-out and fan-in are explicitly named in DEA-C01 Task Statement 1.1 under "Managing fan-in and fan-out for streaming data distribution." These are architectural patterns, not specific services.

**Fan-Out — One source, many consumers:**

Distribute a single data stream or event to multiple independent downstream systems simultaneously.

```
                    ┌──→ Lambda (real-time alerts)
Kinesis Stream ─────┼──→ Firehose (S3 archive)
                    └──→ Managed Flink (aggregations)

SNS Topic ──────────┬──→ SQS Queue A (order service)
                    ├──→ SQS Queue B (inventory service)
                    └──→ Lambda (notification service)
```

**AWS fan-out patterns:**
- **Kinesis + Enhanced Fan-Out:** Each registered consumer (Lambda, Flink, custom KCL app) gets a dedicated 2 MB/s per shard — true parallel consumption without throughput sharing
- **SNS → multiple SQS queues:** Classic fan-out for event-driven microservices. SNS delivers to all subscribed SQS queues simultaneously. Each queue processes independently at its own pace.
- **EventBridge rules:** One event → multiple targets (Lambda, SQS, Step Functions, Kinesis). Rules fan-out based on event pattern matching.

**Fan-In — Many sources, one consumer:**

Aggregate data from multiple independent sources into a single destination or processing layer.

```
Producer A ──┐
Producer B ──┼──→ Kinesis Stream → Lambda (processes all)
Producer C ──┘

IoT Device 1 ──┐
IoT Device 2 ──┼──→ Kinesis Data Stream (single stream, multiple partition keys)
IoT Device 3 ──┘
```

**AWS fan-in patterns:**
- **Kinesis Data Streams with multiple producers:** All producers write to the same stream using different partition keys. One consumer (Lambda/Flink/KCL) reads all records.
- **SQS with multiple producers:** Multiple services enqueue messages to one SQS queue. One consumer Lambda processes all messages.
- **S3 + Glue crawler:** Multiple upstream jobs write Parquet files to the same S3 prefix. One Glue job reads and processes all files.

**Step Functions fan-out with Parallel/Map state:**
```json
{
  "Type": "Parallel",
  "Branches": [
    {"StartAt": "ProcessUSEast"},
    {"StartAt": "ProcessEUWest"},
    {"StartAt": "ProcessAPSouth"}
  ],
  "Next": "AggregateResults"
}
```
Fan-out: process 3 regions in parallel. Fan-in: `AggregateResults` runs after ALL branches complete.

**Exam patterns:**
- "Multiple downstream systems need to receive the same event without coupling the producer to each consumer" → SNS topic (fan-out) with each consumer subscribed as an SQS queue
- "One Lambda must process records from both a Kinesis stream AND a DynamoDB stream" → Two separate Event Source Mappings on the same Lambda function (fan-in to one processor)
- "How to process a Kinesis stream with 3 independent consumers each needing full throughput" → Enable Enhanced Fan-Out, register 3 consumers (fan-out with dedicated bandwidth)

---

## Infrastructure as Code (IaC)

### AWS CloudFormation
- YAML/JSON templates for AWS resource provisioning
- Stack-based deployment and rollback
- Drift detection for configuration changes

### AWS CDK (Cloud Development Kit)
- Write infrastructure in Python, TypeScript, Java, C#
- Synthesizes to CloudFormation templates
- Higher-level constructs for common patterns

### AWS SAM (Serverless Application Model)
- Extension of CloudFormation for serverless
- Package and deploy: Lambda functions, Step Functions, DynamoDB tables, API Gateway
- Local testing with `sam local`

---

## CI/CD Pipeline

```
CodeCommit/GitHub -> CodeBuild (build/test) -> CodeDeploy (deploy) -> CodePipeline (orchestrates all)
```

| Service | Role |
|---------|------|
| **CodePipeline** | Orchestrates the full CI/CD pipeline |
| **CodeBuild** | Build and test code (compile, run unit tests) |
| **CodeDeploy** | Deploy to EC2, Lambda, ECS |

### Git Commands in Scope for DEA-C01

The exam guide explicitly lists specific Git operations under Task Statement 1.4. Know what each command does conceptually — you won't write Git syntax on the exam but you will be asked which command achieves a described goal.

| Command | What It Does | Exam Scenario |
|---|---|---|
| `git clone` | Copy a remote repository to local machine | "Developer needs to work on the pipeline code locally" |
| `git branch feature/new-etl` | Create a new branch | "Isolate development of a new Glue job without affecting main" |
| `git checkout -b` | Create AND switch to a new branch in one command | Standard feature branch workflow |
| `git merge` | Merge a branch into current branch — creates a merge commit | "Integrate completed feature back into main" |
| `git rebase` | Replay commits from one branch onto another — linear history | "Clean up commit history before merging to main" |
| `git cherry-pick <hash>` | Apply a specific commit from another branch to current branch | "Apply a hotfix commit from main to a release branch without merging everything" |
| `git revert <hash>` | Create a new commit that undoes a previous commit — safe for shared branches | "Undo a bad deployment commit without rewriting history" |
| `git stash` | Temporarily shelve uncommitted changes | "Save work-in-progress before switching branches" |

**Branching strategies relevant to data pipeline CI/CD:**
- Feature branches: each new pipeline change in its own branch → PR → merge to main → CodePipeline triggers
- GitOps: infrastructure state (CloudFormation/CDK stacks) stored in Git → changes to repo trigger deployment pipeline automatically
- Exam pattern: "a data engineer needs to test a Glue job change without affecting the production pipeline" → create a feature branch, test in a dev environment, merge to main after approval

---

## Distributed Computing Concepts

- **MapReduce:** Split data across nodes (Map), aggregate results (Reduce)
- **Partitioning:** Distribute data by key for parallel processing
- **Shuffling:** Redistribute data across nodes (expensive in Spark)
- **DAG (Directed Acyclic Graph):** Execution plan for distributed jobs (Spark, Airflow)

---

## AWS Batch — When Lambda and Glue Are Not Enough

AWS Batch is in the official DEA-C01 in-scope services list but is completely absent from most study materials. It appears in exam questions about long-running containerised batch jobs.

**What it is:** Fully managed batch computing service. You define jobs as Docker containers. AWS Batch manages the underlying EC2 or Fargate infrastructure, job queues, and scheduling — you just submit jobs.

**Core concepts:**

| Concept | Description |
|---|---|
| **Job Definition** | Template for a job — specifies Docker image, vCPU, memory, IAM role, retry strategy |
| **Job Queue** | Jobs are submitted to a queue with a priority. Multiple queues can point to one compute environment. |
| **Compute Environment** | Managed (AWS provisions EC2/Fargate) or Unmanaged (you manage EC2 fleet). Supports Spot for cost savings. |
| **Array Jobs** | Submit one job definition that spawns N child jobs in parallel — e.g., process 1,000 files, each as a separate container |

**AWS Batch vs Lambda vs Glue — decision table:**

| Dimension | AWS Lambda | AWS Glue | AWS Batch |
|---|---|---|---|
| **Max runtime** | 15 minutes | 48 hours | No limit (days if needed) |
| **Compute type** | Serverless (managed) | Serverless Spark (managed) | EC2 or Fargate (managed provisioning) |
| **Custom runtime/libraries** | Limited (layers, container up to 10 GB) | Limited (custom JARs, Python libs) | Any Docker image — full control |
| **Workload type** | Event-driven, lightweight | ETL/ELT, Spark/Python Shell | Long-running batch, simulations, ML training, custom tools |
| **Spark support** | No | Yes (native) | Only if you containerise Spark yourself |
| **Cost model** | Per 1ms invocation | Per DPU-second | Per EC2/Fargate instance-hour (Spot available) |

**Exam patterns:**
- "Run a containerised Python script that processes genomics data for 6 hours — Lambda is too short, Glue doesn't support this runtime" → AWS Batch
- "Nightly batch job needs to run a custom Fortran simulation in a Docker container for up to 12 hours" → AWS Batch (Lambda max 15 min, Glue is Spark/Python only)
- "Process 500 independent data files in parallel using the same container image" → AWS Batch Array Job (spawns 500 child jobs, one per file)
- "Most cost-effective way to run AWS Batch jobs that can tolerate interruption" → Managed Compute Environment with Spot instances

**AWS Batch in a data pipeline:**
```
EventBridge (schedule) → Step Functions → AWS Batch (submit job) → S3 (output)
                                       ↕ .sync integration
                                  waits for Batch job to complete
```

Step Functions has native `.sync` integration with AWS Batch — Step Functions submits the job and waits for completion before moving to the next state.

---

## Data Structures (In Scope)

- **Graph structures:** Used in Neptune, knowledge graphs, lineage tracking
- **Tree structures:** Used in hierarchical data, JSON/XML parsing
- **Hash tables:** DynamoDB partition key hashing for data distribution

---

## Exam Gotchas

- Know that CDK synthesizes to CloudFormation (not a replacement, an abstraction)
- SAM is specifically for serverless deployments
- "Repeatable infrastructure deployment" = CloudFormation or CDK
- "Package and deploy Lambda" = SAM
- Version control, testing, and logging are expected best practices
- Understand distributed computing concepts (don't need to write code)
- **ETL vs ELT is an architectural choice, not just a terminology difference:** The exam gives scenarios and asks which approach is appropriate. Key signal: if the question mentions "the destination (Redshift/Athena) has sufficient compute" or "preserve raw data for future reprocessing" → lean ELT. If "reduce data volume before loading" or "destination has limited compute" → lean ETL.
- **Idempotency is not the same as exactly-once delivery:** Exactly-once means a message is delivered exactly one time (hard to guarantee in distributed systems). Idempotency means processing the same message multiple times produces the same result. Design your pipelines to be idempotent — assume you will receive duplicates and handle them gracefully.
- **Partition pruning requires the WHERE clause to use the partition column in its raw form:** Wrapping a partition column in a function (CAST, TO_DATE, LOWER, etc.) breaks partition pruning in Athena and Spark, causing a full table scan. Always store partition columns in the exact type and format they will be queried in.
- **AWS CDK synthesizes to CloudFormation — it is not a replacement:** CDK is an abstraction layer. When you run `cdk deploy`, it first runs `cdk synth` to generate a CloudFormation template, then deploys that template. Understanding this matters for exam questions about rollback, drift detection, and stack operations — those are CloudFormation concepts that apply equally to CDK-deployed stacks.
- **SAM is CloudFormation with shortcuts for serverless:** SAM templates are valid CloudFormation templates with additional resource types (AWS::Serverless::Function, AWS::Serverless::Api). `sam build` and `sam deploy` ultimately call CloudFormation. SAM is the right answer when the question asks about packaging and deploying Lambda functions, Step Functions, and API Gateway together as a unit.
- **Distributed Map in Step Functions vs EMR for large-scale parallel processing:** Both can process millions of items in parallel. Decision signal: if items are AWS API calls or Lambda invocations → Distributed Map (Step Functions). If items are large data files needing Spark/Hive processing → EMR. If items are containerised scripts → AWS Batch Array Jobs.
