# AWS Lambda

Serverless compute service that runs code in response to events.

---

## Key Specs

| Limit | Value |
|-------|-------|
| **Timeout** | Max 15 minutes |
| **Memory** | 128 MB to 10 GB |
| **Storage** | /tmp = 512 MB (up to 10 GB) |
| **Deployment Package** | 50 MB (zipped), 250 MB (unzipped) |
| **Concurrency** | 1,000 default (can request increase) |
| **Layers** | Up to 5 layers per function |

---

## Invocation Types

| Type | Description | Use Case |
|------|-------------|----------|
| **Synchronous** | Caller waits for response | API Gateway, SDK calls |
| **Asynchronous** | Fire-and-forget, retries on failure | S3 events, SNS, EventBridge |
| **Event Source Mapping** | Polls from stream/queue | Kinesis, SQS, DynamoDB Streams |

---

## Data Engineering Use Cases

- **S3 Event Processing:** File arrives -> Lambda validates/transforms -> writes result
- **Kinesis Consumer:** Process stream records in micro-batches
- **Firehose Transformation:** Inline transformation of streaming data before delivery
- **Lightweight ETL:** Small file conversions, header validation
- **Orchestration Glue:** Trigger Glue jobs, call APIs between steps

---

## Concurrency

- **Reserved Concurrency:** Guarantees a set number of instances for a function
- **Provisioned Concurrency:** Pre-initializes instances to avoid cold starts
- Cold start: 100ms - 10s depending on runtime and VPC configuration
- **VPC Lambda:** Historically slow cold start, now uses Hyperplane ENI (much faster)

---

## Lambda with EFS

- Mount EFS file system for shared, persistent storage across invocations
- Use when /tmp (10 GB) is insufficient
- Requires Lambda to be in a VPC

---

## Exam Gotchas

- **15-minute timeout** is the key constraint. If processing takes longer, use Glue or EMR
- Lambda is the answer for "event-driven, lightweight, sporadic" processing
- Lambda is NOT for heavy ETL (use Glue/EMR)
- Lambda + DLQ (Dead Letter Queue) for handling failed async invocations
- Lambda Layers for shared libraries across functions
- Use **Provisioned Concurrency** to avoid cold starts for latency-sensitive applications
