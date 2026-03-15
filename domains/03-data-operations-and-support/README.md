# Domain 3: Data Operations and Support (22%)

Covers automating data processing, analyzing data, maintaining/monitoring pipelines, and ensuring data quality.

## Task Statements

### Task 3.1: Automate Data Processing
- Orchestrate data pipelines (MWAA, Step Functions)
- Troubleshoot managed workflows
- Call SDKs to access AWS features from code
- Process data (EMR, Redshift, Glue)
- Consume and maintain data APIs
- Prepare data for transformation (Glue DataBrew, SageMaker Unified Studio)
- Query data (Athena)
- Automate with Lambda
- Manage events and schedulers (EventBridge)

### Task 3.2: Analyze Data
- Visualize data (DataBrew, QuickSight)
- Verify and clean data (Lambda, Athena, QuickSight, Jupyter Notebooks, SageMaker Data Wrangler)
- SQL in Redshift and Athena (queries and views)
- Athena notebooks with Apache Spark
- Tradeoffs: provisioned vs serverless
- Data aggregation, rolling average, grouping, pivoting

### Task 3.3: Maintain and Monitor Pipelines
- Extract logs for audits
- Deploy logging and monitoring solutions
- Use notifications for alerts
- Troubleshoot performance issues
- CloudTrail for API tracking
- Troubleshoot Glue and EMR pipelines
- CloudWatch Logs configuration
- Analyze logs (Athena, EMR, OpenSearch, CloudWatch Logs Insights)

### Task 3.4: Ensure Data Quality
- Run quality checks during processing (empty field checks)
- Define data quality rules (DataBrew)
- Investigate data consistency
- Describe data sampling techniques
- Implement data skew mechanisms

## Study Files

| Topic | File |
|-------|------|
| Orchestration (Step Functions, MWAA, EventBridge) | [orchestration.md](./orchestration.md) |
| CloudWatch (Metrics, Logs, Alarms, Dashboards) | [monitoring-cloudwatch.md](./monitoring-cloudwatch.md) |
| CloudTrail (Auditing, Data Events) | [aws-cloudtrail.md](./aws-cloudtrail.md) |
| Amazon Athena (Serverless SQL) | [amazon-athena.md](./amazon-athena.md) |
| Amazon QuickSight (Visualization) | [amazon-quicksight.md](./amazon-quicksight.md) |
| Data Quality (Glue DQ, DataBrew) | [data-quality.md](./data-quality.md) |
| Error Handling (DLQ, Retries, Backoff) | [error-handling.md](./error-handling.md) |
| Cost Optimization | [cost-optimization.md](./cost-optimization.md) |
| Performance Tuning (Spark, Redshift, Kinesis) | [performance-tuning.md](./performance-tuning.md) |
