# Domain 3: Data Operations and Support (22%)

Covers automating data processing, analyzing data, maintaining/monitoring pipelines, and ensuring data quality.

---

## Task Statements

### Task 3.1: Automate Data Processing
- Orchestrate data pipelines (MWAA, Step Functions, Glue Workflows, EventBridge)
- Troubleshoot managed workflows
- Call SDKs to access AWS features from code
- Process data (EMR, Redshift, Glue)
- Consume and maintain data APIs
- Prepare data for transformation (Glue DataBrew, SageMaker Unified Studio)
- Query data (Athena, including Iceberg DML via Athena SQL)
- Automate with Lambda (Lambda Destinations for async success/failure routing)
- Manage events and schedulers (EventBridge, EventBridge Scheduler)
- Large-scale parallel processing (Step Functions Distributed Map — up to 10M items from S3)

### Task 3.2: Analyze Data
- Visualize data (DataBrew, QuickSight)
- Verify and clean data (Lambda, Athena, QuickSight, Jupyter Notebooks, SageMaker Data Wrangler)
- SQL in Redshift and Athena (queries, views, CTAS, UNLOAD)
- Athena notebooks with Apache Spark
- Tradeoffs: provisioned vs serverless
- Data aggregation, rolling average, grouping, pivoting
- Row-level security for multi-tenant dashboards (QuickSight RLS)

### Task 3.3: Maintain and Monitor Pipelines
- Extract logs for audits (CloudTrail, CloudWatch Logs)
- Deploy logging and monitoring solutions (CloudWatch Metrics, Alarms, Subscription Filters, EMF)
- Use notifications for alerts (SNS, CloudWatch Alarms, Composite Alarms)
- Troubleshoot performance issues (Glue, EMR, Redshift, Kinesis, Lambda)
- CloudTrail for API tracking (Management Events, Data Events, Insights, Lake)
- CloudWatch Logs configuration (retention, encryption, metric filters)
- Emit metrics without PutMetricData API overhead (CloudWatch EMF via structured logs)
- Analyze logs (Athena, CloudWatch Logs Insights, CloudTrail Lake)
- Distributed tracing with X-Ray

### Task 3.4: Ensure Data Quality
- Run quality checks during processing (Glue Data Quality, DQDL rules)
- Define data quality rules (DataBrew Profile Jobs, Glue DQDL)
- Investigate data consistency (referential integrity, freshness, volume checks)
- Describe data sampling techniques
- Implement data skew mechanisms
- Handle schema evolution and compatibility

---

## Study Files

| Topic | File | Key Concepts Added |
|-------|------|--------------------|
| Orchestration | [orchestration.md](./orchestration.md) | Step Functions state types, `.sync:2` / `.waitForTaskToken` patterns, Distributed Map (10M items from S3), input/output path processing, MWAA operators, EventBridge Archive/Replay, Pipes |
| CloudWatch Monitoring | [monitoring-cloudwatch.md](./monitoring-cloudwatch.md) | Subscription filters, custom metrics (PutMetricData), EMF (Embedded Metric Format), Metric Math, Anomaly Detection, Logs Insights queries, TreatMissingData, CloudWatch Synthetics |
| CloudTrail Auditing | [aws-cloudtrail.md](./aws-cloudtrail.md) | Organization trails, CloudTrail Lake SQL queries, Athena on CloudTrail, log integrity validation, KMS encryption, SCP protection |
| Amazon Athena | [amazon-athena.md](./amazon-athena.md) | Engine v3 vs v2, workgroups (byte limits), partition projection types, result reuse, prepared statements, CTAS/UNLOAD, federated query, Iceberg DML (MERGE INTO, UPDATE, DELETE, OPTIMIZE, VACUUM) |
| Amazon QuickSight | [amazon-quicksight.md](./amazon-quicksight.md) | SPICE vs Direct Query, RLS / tag-based RLS, column-level security, ML insights, QuickSight Q, embedding (signed URLs), Author vs Reader pricing |
| Data Quality | [data-quality.md](./data-quality.md) | DQDL deep dive, QUARANTINE/FAIL/LOG actions, DataBrew (projects, recipes, profile jobs), schema evolution, referential integrity patterns |
| Error Handling | [error-handling.md](./error-handling.md) | Step Functions Retry/Catch syntax, Lambda Destinations (vs DLQ), Kinesis bisectBatch, SQS DLQ patterns, idempotency implementations, circuit breaker, X-Ray tracing, error code reference |
| Cost Optimization | [cost-optimization.md](./cost-optimization.md) | Compute Optimizer, RI vs Savings Plans, S3 storage classes, NAT Gateway vs VPC endpoints, unit economics, Cost Anomaly Detection, Athena vs Redshift vs EMR TCO |
| Performance Tuning | [performance-tuning.md](./performance-tuning.md) | AQE (Adaptive Query Execution), Glue worker types + job bookmarks, EMR Instance Fleets + Managed Scaling, Redshift EXPLAIN + WLM (interleaved sort key deprecated), Lambda cold starts + connection pooling |

---

## Key Service Cross-Reference

| Service | Primary Task | File(s) |
|---------|-------------|---------|
| Step Functions | Orchestration, error handling | orchestration.md, error-handling.md |
| MWAA (Airflow) | Complex DAG orchestration, backfilling | orchestration.md |
| EventBridge | Event routing, scheduling | orchestration.md |
| Glue Workflows | Glue-only orchestration | orchestration.md |
| CloudWatch | Metrics, alarms, logs, dashboards | monitoring-cloudwatch.md |
| CloudTrail | API auditing, compliance | aws-cloudtrail.md |
| Athena | Serverless SQL, data lake queries | amazon-athena.md |
| QuickSight | BI dashboards, visualization | amazon-quicksight.md |
| Glue Data Quality | DQDL rules, quality gates in ETL | data-quality.md |
| Glue DataBrew | No-code prep, data profiling | data-quality.md |
| Lambda | Automation, event processing | error-handling.md, performance-tuning.md |
| EMR | Large-scale Spark/Hadoop | performance-tuning.md |
| Kinesis | Streaming, consumer lag | performance-tuning.md, error-handling.md |
| Redshift | OLAP queries, WLM, tuning | performance-tuning.md |
| X-Ray | Distributed tracing | error-handling.md |

---

## Exam Quick-Picks

| Scenario keyword | Answer |
|-----------------|--------|
| "Serverless SQL on S3" | Athena |
| "BI dashboard / visualization" | QuickSight |
| "Backfill historical data by date range" | MWAA (Airflow) |
| "Wait for Glue job to finish in Step Functions" | `.sync:2` integration pattern |
| "Human approval step in pipeline" | Step Functions `.waitForTaskToken` |
| "No-code data preparation" | Glue DataBrew |
| "Data quality rules in ETL pipeline" | Glue Data Quality (DQDL) |
| "Isolate bad records without stopping pipeline" | QUARANTINE action in Glue DQ |
| "Who deleted this S3 object?" | CloudTrail S3 Data Events |
| "Query CloudTrail without S3 + Athena setup" | CloudTrail Lake |
| "Unusual API activity detection" | CloudTrail Insights |
| "Real-time log streaming from CloudWatch" | Subscription Filters |
| "Ad-hoc log analysis" | CloudWatch Logs Insights |
| "EC2 memory / disk metrics" | CloudWatch Agent (not a default metric) |
| "Prevent runaway Athena query costs" | Workgroup per-query bytes scanned limit |
| "Multi-tenant dashboard — each user sees only their data" | QuickSight Row-Level Security (RLS) |
| "Anonymous embed with per-tenant filtering" | Tag-based RLS |
| "Kinesis consumer falling behind" | High IteratorAge → add shards or increase batch size |
| "Poison-pill record blocking Kinesis shard" | `bisectBatchOnFunctionError` + `maximumRecordAgeInSeconds` |
| "Handle failed messages without blocking" | Dead Letter Queue (SQS DLQ) |
| "Prevent same message processed twice" | Idempotency (conditional write / SQS FIFO dedup) |
| "Distributed tracing across Lambda + DynamoDB + SQS" | AWS X-Ray |
| "Detect unexpected cost spikes" | AWS Cost Anomaly Detection |
| "Optimize Athena cost" | Parquet + partitioning + result reuse |
| "Step Functions high-volume short executions" | Express Workflows |
| "Process millions of S3 records in parallel with Step Functions" | Distributed Map (Express child workflows) |
| "Route Lambda async success AND failure to different queues" | Lambda Destinations |
| "Emit metrics from Lambda without extra API calls" | CloudWatch EMF |
| "Delete/update specific rows in a data lake using SQL" | Athena + Iceberg (MERGE INTO / DELETE) |
| "Compact small files in Iceberg table without Spark" | Athena `OPTIMIZE ... REWRITE DATA` |
