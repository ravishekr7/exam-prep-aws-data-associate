# Amazon CloudWatch

Monitoring and observability service for AWS resources and applications.

---

## Components

### Metrics
Numerical data points for AWS resources:
- **EC2:** CPU%, Network, Disk
- **Glue:** Job Duration, DPU Hours, Bytes Read/Written
- **Kinesis:** IncomingBytes, IteratorAge, ReadProvisionedThroughputExceeded
- **Lambda:** Invocations, Duration, Errors, Throttles, ConcurrentExecutions
- **Redshift:** CPU%, Disk Space, Query Duration

### Alarms
Trigger actions when metrics cross thresholds:
- **States:** OK, ALARM, INSUFFICIENT_DATA
- **Actions:** SNS notification, Auto Scaling, Lambda, EC2 actions
- **Period:** Minimum 60 seconds (1-second with detailed monitoring)
- **Composite Alarms:** Combine multiple alarms with AND/OR logic

### Logs
- **Log Groups:** Collection of log streams (e.g., `/aws/lambda/my-function`)
- **Log Streams:** Individual log outputs
- **Retention:** Configurable (1 day to never expire)
- **Metric Filters:** Extract metric values from log data

### CloudWatch Logs Insights
- Interactive query language for searching and analyzing logs
- SQL-like syntax: `fields @timestamp, @message | filter @message like /ERROR/ | sort @timestamp desc`
- Use for ad-hoc log analysis

### Dashboards
- Visualize metrics in real-time
- Cross-account and cross-region dashboards
- Automatic and custom dashboards

---

## Key Metrics for Data Engineering

| Service | Metric | What It Tells You |
|---------|--------|-------------------|
| **Kinesis** | `IteratorAge` | How far behind the consumer is (lag) |
| **Kinesis** | `ReadProvisionedThroughputExceeded` | Need more shards |
| **Glue** | `glue.driver.aggregate.bytesRead` | Data volume processed |
| **Lambda** | `Throttles` | Hitting concurrency limit |
| **Redshift** | `PercentageDiskSpaceUsed` | Running out of storage |

---

## Exam Gotchas

- CloudWatch Logs Insights for **ad-hoc log analysis**
- CloudWatch Alarms + SNS for **alerting**
- **IteratorAge** is the key Kinesis health metric
- Metric Filters extract custom metrics from log text
- Detailed monitoring (1-second) costs extra vs basic (5-minute)
