# Amazon CloudWatch

Monitoring and observability service for AWS resources and applications.

---

## Components Overview

| Component | Purpose |
|-----------|---------|
| **Metrics** | Numerical time-series data points from AWS services |
| **Alarms** | Trigger actions when metrics breach thresholds |
| **Logs** | Collect, store, and analyze log data |
| **Logs Insights** | Interactive query language for log analysis |
| **Dashboards** | Visualize metrics and alarms in real time |
| **Subscription Filters** | Stream logs in real time to Lambda/Kinesis |
| **Metric Filters** | Extract custom metrics from log data |
| **Anomaly Detection** | ML-based automatic baseline for metrics |
| **Synthetics** | Canary scripts for endpoint monitoring |
| **Container Insights** | Enhanced metrics for ECS/EKS/Kubernetes |

---

## Metrics

### Metric Resolution

| Type | Resolution | Cost |
|------|-----------|------|
| **Standard** | 1-minute granularity | Free tier included |
| **High-resolution** | 1-second granularity | Extra cost via `PutMetricData` |
| **Basic monitoring** | 5-minute (default EC2) | Free |
| **Detailed monitoring** | 1-minute (opt-in for EC2) | Small extra cost |

### Custom Metrics via PutMetricData

Publish application-level metrics directly from code:

```python
import boto3
cloudwatch = boto3.client('cloudwatch')

cloudwatch.put_metric_data(
    Namespace='MyApp/ETL',
    MetricData=[{
        'MetricName': 'RecordsProcessed',
        'Value': 15000,
        'Unit': 'Count',
        'Dimensions': [
            {'Name': 'JobName', 'Value': 'daily-ingestion'}
        ]
    }]
)
```

- **Namespace:** Logical grouping (e.g., `MyApp/ETL`)
- **Dimensions:** Key-value pairs that identify a unique metric stream
- Custom metrics are retained for 15 months

### CloudWatch Embedded Metric Format (EMF)

EMF allows Lambda (and other services) to emit custom CloudWatch metrics by writing **structured JSON log lines** — without calling `PutMetricData`. CloudWatch automatically extracts the metrics from the log output.

**Why EMF instead of PutMetricData:**
- `PutMetricData` is a separate API call — adds latency inside the Lambda handler and counts against CloudWatch API quotas
- EMF piggybacks on the existing log stream — zero additional API calls, zero added latency
- Works in Lambda, ECS, EC2, and any service that writes to CloudWatch Logs

**EMF log format:**

```python
import json

def handler(event, context):
    records_processed = process_batch(event)

    # EMF structured log line — CloudWatch auto-extracts the metric
    print(json.dumps({
        "_aws": {
            "Timestamp": 1705312800000,
            "CloudWatchMetrics": [{
                "Namespace": "MyApp/ETL",
                "Dimensions": [["JobName"]],
                "Metrics": [
                    {"Name": "RecordsProcessed", "Unit": "Count"},
                    {"Name": "ProcessingDurationMs", "Unit": "Milliseconds"}
                ]
            }]
        },
        "JobName": "daily-ingestion",           # dimension value
        "RecordsProcessed": records_processed,  # metric value
        "ProcessingDurationMs": 1234            # metric value
    }))
```

- CloudWatch Logs detects the `_aws.CloudWatchMetrics` key and automatically creates the metrics
- The same log line also appears as a normal log event — no data lost
- Use the **`aws-embedded-metrics` library** (Python/Node.js) for a cleaner API:

```python
from aws_embedded_metrics import metric_scope

@metric_scope
def handler(event, context, metrics):
    metrics.set_namespace("MyApp/ETL")
    metrics.put_dimensions({"JobName": "daily-ingestion"})
    metrics.put_metric("RecordsProcessed", 15000, "Count")
    metrics.put_metric("ProcessingDurationMs", 1234, "Milliseconds")
```

**Exam Patterns:**
- "Emit custom business metrics from Lambda without adding API call latency or hitting PutMetricData quotas" → CloudWatch EMF
- "Lambda function needs to publish high-frequency metrics cheaply without extra AWS API calls" → EMF (metrics emitted via logs, not PutMetricData)
- "Which approach adds the least overhead when emitting metrics from a high-concurrency Lambda?" → EMF over PutMetricData

### Metric Math

Combine metrics using mathematical expressions:

```
# Error rate percentage
m1 = Lambda Errors
m2 = Lambda Invocations
error_rate = (m1 / m2) * 100

# Kinesis utilization
put_utilization = PUT_RECORDS_BYTES / (SHARD_COUNT * 1048576) * 100
```

- Use in dashboards and alarms
- Supports functions: `SUM`, `AVG`, `MIN`, `MAX`, `RATE`, `DIFF`, `FILL`, `RUNNING_SUM`
- Can reference metrics from different accounts/regions in a single expression

### Anomaly Detection

- CloudWatch learns the **normal baseline** of a metric using ML (considers time of day, day of week)
- Creates a dynamic band (upper/lower bounds) around expected values
- **Alarm on Anomaly:** Alert when metric goes outside the band
- No threshold to manually configure — adapts to seasonal patterns automatically
- Apply to any metric with sufficient history (2 weeks recommended)

> **Exam tip:** Use Anomaly Detection for metrics that fluctuate on a schedule (e.g., daily traffic patterns). Use static thresholds for metrics with fixed limits (e.g., disk usage).

---

## Key Metrics for Data Engineering

| Service | Metric | What It Tells You |
|---------|--------|-------------------|
| **Kinesis** | `IteratorAge` | Consumer lag — how far behind (milliseconds) |
| **Kinesis** | `ReadProvisionedThroughputExceeded` | Need more shards or Enhanced Fan-Out |
| **Kinesis** | `WriteProvisionedThroughputExceeded` | Producers hitting shard write limits |
| **Kinesis** | `IncomingBytes` | Volume of data entering the stream |
| **Glue** | `glue.driver.aggregate.bytesRead` | Data volume processed |
| **Glue** | `glue.driver.aggregate.recordsRead` | Record count processed |
| **Glue** | `glue.ALL.jvm.heap.usage` | Memory pressure on Glue workers |
| **Lambda** | `Throttles` | Hitting concurrency limit — need limit increase |
| **Lambda** | `ConcurrentExecutions` | Current parallel execution count |
| **Lambda** | `Duration` | Execution time — watch for timeouts |
| **Lambda** | `Iterator Age` | Kinesis/DynamoDB stream consumer lag |
| **Redshift** | `PercentageDiskSpaceUsed` | Running out of storage |
| **Redshift** | `QueryDuration` | Slow query detection |
| **Redshift** | `CPUUtilization` | Compute pressure |
| **EMR** | `HDFSUtilization` | HDFS storage usage |
| **S3** | `5xxErrors` | S3 server-side errors |
| **DynamoDB** | `ThrottledRequests` | Provisioned throughput exceeded |
| **DynamoDB** | `ConsumedReadCapacityUnits` | Track against provisioned RCUs |

---

## Alarms

### Alarm States

- **OK:** Metric within threshold
- **ALARM:** Metric has breached threshold
- **INSUFFICIENT_DATA:** Not enough data points yet

### Alarm Configuration

```
Period × Evaluation Periods = window of time evaluated

Example: Period=60s, EvaluationPeriods=5, DatapointsToAlarm=3
→ Out of the last 5 minutes, if 3 or more are in ALARM → trigger
```

- **DatapointsToAlarm:** Minimum data points that must breach (M out of N evaluation)
- **TreatMissingData:** How to handle gaps in data:
  - `missing` (default) — alarm stays in current state
  - `notBreaching` — treat missing as OK
  - `breaching` — treat missing as ALARM
  - `ignore` — don't change alarm state

> **Exam tip:** For metrics that stop reporting when a resource is down (e.g., an EC2 instance that terminates), set `TreatMissingData=breaching` so the alarm fires when data disappears.

### Alarm Actions

- **SNS notification** (email, SMS, Lambda trigger, SQS)
- **Auto Scaling** (scale in/out EC2, ECS, DynamoDB)
- **EC2 actions** (stop, terminate, reboot, recover)
- **Lambda** (via SNS subscription)
- **Systems Manager OpsCenter** (create OpsItem)

### Composite Alarms

- Combine multiple alarms with **AND / OR** logic
- Example: Alert only when `CPU > 90%` AND `DiskUsage > 80%`
- Reduces alert noise from correlated alarms
- Composite alarms do not perform actions directly — they trigger based on child alarm states

---

## CloudWatch Logs

### Structure

```
Log Group (e.g., /aws/lambda/my-function)
  └── Log Stream (e.g., 2024/01/15/[$LATEST]abc123)
        └── Log Events (individual log lines with timestamp)
```

### Log Group Configuration

| Setting | Options | Notes |
|---------|---------|-------|
| **Retention** | 1 day to Never | Default: Never (costs money) — always set retention |
| **Encryption** | AWS-managed or KMS CMK | KMS encryption for compliance |
| **Resource policy** | Cross-account access | Allow other accounts to write logs |

### Metric Filters

Extract numeric values from log text and publish as CloudWatch Metrics:

```
Filter pattern:  [timestamp, requestId, level="ERROR", ...]
Metric:          ErrorCount (increment by 1 on each match)
```

- Applied to a log group
- Metric filters only process new log events (not historical)
- Up to 100 metric filters per log group

### Subscription Filters

Route log data in **real time** to processing services:

| Destination | Use case |
|-------------|---------|
| **Lambda** | Real-time log processing, alerting, enrichment |
| **Kinesis Data Streams** | High-volume streaming to downstream consumers |
| **Kinesis Firehose** | Real-time delivery to S3, Redshift, OpenSearch |
| **OpenSearch Service** | Real-time log indexing and search |

```
Log Group → Subscription Filter (pattern) → Kinesis Stream → Lambda → Alert
```

- Each log group supports **1 subscription filter** for Kinesis/Firehose, **2 for Lambda**
- Filter pattern narrows which log events are forwarded (e.g., only ERROR lines)
- **Cross-account log aggregation:** Send logs from multiple accounts to a central Kinesis stream using subscription filters + resource policies

> **Exam tip:** Subscription filters = real-time log processing. Logs Insights = ad-hoc analysis. Know the difference.

---

## CloudWatch Logs Insights

Interactive query language for analyzing log data.

### Query Syntax

```sql
# Basic error search
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 20

# Count errors by type over time
fields @timestamp, @message
| parse @message "ERROR: *" as errorType
| stats count(*) as errorCount by errorType
| sort errorCount desc

# P99 Lambda duration
filter @type = "REPORT"
| stats pct(@duration, 99) as p99_duration,
        avg(@duration) as avg_duration
        by bin(5m)

# Find Lambda cold starts
filter @type = "REPORT"
| filter @initDuration > 0
| stats count(*) as coldStarts by bin(1h)

# Top 10 slowest queries in Athena
fields @timestamp, queryExecutionId, statistics.dataScannedInBytes
| sort statistics.dataScannedInBytes desc
| limit 10
```

### Lambda Built-in Fields

CloudWatch automatically parses Lambda log fields:
- `@requestId` — Lambda invocation ID
- `@duration` — execution time (ms)
- `@billedDuration` — rounded up duration
- `@memorySize` — configured memory
- `@maxMemoryUsed` — peak memory used
- `@initDuration` — cold start duration (only present on cold starts)

### Best Practices

- Use **structured logging (JSON)** — Logs Insights can parse JSON fields automatically
- Use `parse` command for unstructured log formats
- Use `stats ... by bin(5m)` for time-series aggregations
- Save frequent queries as **Saved Queries**

---

## CloudWatch Dashboards

- Visualize metrics, alarms, and Logs Insights queries in one place
- **Cross-account and cross-region dashboards** — aggregate view across AWS accounts
- Automatic refresh (10s, 1m, 2m, 5m, 15m)
- Share dashboards publicly (snapshot URL) or with specific IAM users

---

## CloudWatch Synthetics (Canary Testing)

- Run **canary scripts** (Node.js or Python) on a schedule against your endpoints
- Monitors availability, latency, and broken workflows
- Built-in blueprints: heartbeat monitor, API canary, broken link checker, visual monitoring
- Results stored in S3; metrics in CloudWatch; screenshots for visual tests
- Use case: Monitor Athena query endpoints, API Gateway, or S3 presigned URL availability

---

## CloudWatch Container Insights

- Enhanced metrics and logs for **ECS, EKS, and Kubernetes on EC2**
- Metrics: CPU, memory, disk, network at cluster/service/task level
- Collects logs from containers via the CloudWatch agent or Fluent Bit
- Performance dashboards available out-of-the-box in CloudWatch console

---

## CloudWatch Agent

- Installed on EC2 or on-premises servers to collect:
  - **System metrics:** Memory, disk usage (not collected by default)
  - **Application logs:** Any log file on the instance
- Configuration via SSM Parameter Store (centrally managed)
- Required for memory utilization alarms on EC2 (not a default metric)

> **Exam tip:** If a question asks about EC2 memory monitoring or disk usage, the answer involves installing the CloudWatch agent.

---

## Common Patterns

### Pattern 1: Pipeline Failure Alerting

```
Glue Job Failure → CloudWatch Metric (JobRunFailed)
                → CloudWatch Alarm → SNS → Email/PagerDuty
```

### Pattern 2: Real-Time Error Detection

```
Lambda Logs → Subscription Filter (ERROR pattern) → Lambda → SNS Alert
```

### Pattern 3: Log Aggregation (Multi-Account)

```
Account A Logs → Subscription Filter → Kinesis (central account)
Account B Logs → Subscription Filter ↗
Account C Logs → Subscription Filter ↗
                                        → Firehose → S3 (central log archive)
```

### Pattern 4: Kinesis Lag Monitoring

```
Kinesis IteratorAge metric → CloudWatch Alarm (> 60 seconds)
                           → SNS → trigger Lambda to add shards
```

### Pattern 5: Custom Business Metric

```
Lambda processes order → PutMetricData("Orders/Completed", 1)
                       → CloudWatch Metric → Dashboard → Alarm if drops to 0
```

---

## Exam Gotchas

- **CloudWatch Logs Insights** = ad-hoc analysis. **Subscription Filters** = real-time streaming
- **Metric Filters** only process new log events — not historical data
- **EC2 memory/disk metrics require the CloudWatch agent** — not available by default
- **`TreatMissingData=breaching`** — use when a metric stopping means something is wrong
- **Composite Alarms** reduce alert noise — combine multiple alarm states with AND/OR
- **Anomaly Detection** adapts to seasonal patterns — no manual threshold needed
- **Log Group retention is Never by default** — always set a retention policy to control costs
- **Subscription filter limit:** 1 per log group for Kinesis/Firehose, 2 for Lambda
- **High-resolution metrics** (1-second) are for custom metrics only — not all AWS services publish at 1s
- **Metric Math** can compute derived metrics (error rate, utilization %) without custom PutMetricData calls
- CloudWatch **does not monitor on-premises** resources by default — requires CloudWatch agent
- **EMF (Embedded Metric Format) vs PutMetricData:** EMF emits metrics via structured log lines — zero extra API calls, zero added latency inside the function. Use EMF for high-frequency or high-concurrency Lambda functions where PutMetricData API overhead is a concern.
