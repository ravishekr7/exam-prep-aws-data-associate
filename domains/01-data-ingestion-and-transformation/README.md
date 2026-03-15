# Domain 1: Data Ingestion and Transformation (34%)

The highest-weighted domain. Covers ingesting data from batch and streaming sources, transforming it, orchestrating pipelines, and applying programming concepts.

## Task Statements

### Task 1.1: Perform Data Ingestion
- Read data from streaming sources (Kinesis, MSK, DynamoDB Streams, DMS, Glue, Redshift)
- Read data from batch sources (S3, Glue, EMR, DMS, Redshift, Lambda, AppFlow)
- Configure batch ingestion options
- Consume data APIs
- Set up schedulers (EventBridge, Airflow, time-based schedules)
- Set up event triggers (S3 Event Notifications, EventBridge)
- Call Lambda from Kinesis
- Implement throttling and rate limit handling (DynamoDB, RDS, Kinesis)
- Manage fan-in/fan-out for streaming data
- Understand replayability of pipelines
- Define stateful vs stateless transactions

### Task 1.2: Transform and Process Data
- Optimize container usage (EKS, ECS)
- Connect to data sources (JDBC, ODBC)
- Integrate data from multiple sources
- Optimize costs while processing data
- Choose transformation services (EMR, Glue, Lambda, Redshift)
- Transform between formats (CSV to Parquet, etc.)
- Troubleshoot transformation failures and performance issues
- Create data APIs
- Understand volume, velocity, and variety
- Integrate LLMs for data processing

### Task 1.3: Orchestrate Data Pipelines
- Build workflows (Lambda, EventBridge, MWAA, Step Functions, Glue workflows)
- Design for performance, availability, scalability, resiliency, fault tolerance
- Implement serverless workflows
- Use notification services (SNS, SQS)

### Task 1.4: Apply Programming Concepts
- Optimize code for performance
- Configure Lambda concurrency and performance
- Use programming languages (Python, SQL, Scala, R, Java, Bash, PowerShell)
- Apply software engineering best practices (version control, testing, logging)
- Use IaC (CloudFormation, CDK, SAM)
- Describe CI/CD, distributed computing, data structures and algorithms

## Study Files

| Service/Topic | File |
|--------------|------|
| AWS Glue (Catalog, Crawlers, ETL, DataBrew) | [aws-glue.md](./aws-glue.md) |
| Amazon Kinesis (KDS, Firehose, Flink) | [amazon-kinesis.md](./amazon-kinesis.md) |
| AWS DMS + Schema Conversion Tool | [aws-dms.md](./aws-dms.md) |
| Amazon S3 Ingestion Patterns | [amazon-s3-ingestion.md](./amazon-s3-ingestion.md) |
| Amazon EMR | [amazon-emr.md](./amazon-emr.md) |
| Amazon MSK (Managed Kafka) | [amazon-msk.md](./amazon-msk.md) |
| AWS Step Functions | [aws-step-functions.md](./aws-step-functions.md) |
| Amazon MWAA (Managed Airflow) | [amazon-mwaa.md](./amazon-mwaa.md) |
| AWS Lambda | [aws-lambda.md](./aws-lambda.md) |
| Amazon AppFlow | [amazon-appflow.md](./amazon-appflow.md) |
| Programming Concepts + IaC | [programming-concepts.md](./programming-concepts.md) |
