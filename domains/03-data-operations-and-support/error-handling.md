# Error Handling and Reliability

Patterns for building resilient data pipelines.

---

## Dead Letter Queues (DLQ)

A side-queue (SQS) where failed messages are sent after N retries.

### Supported By
- **Lambda:** Failed async invocations -> DLQ (SQS or SNS)
- **SQS:** Messages that exceed maxReceiveCount -> DLQ
- **SNS:** Failed deliveries -> DLQ

### Pattern
```
Source -> SQS Queue -> Lambda (fails 3x) -> DLQ -> Manual review or reprocessing
```

---

## Retry Strategies

### Exponential Backoff
When throttled ("Rate Exceeded"):
- Wait 1s, retry -> fail
- Wait 2s, retry -> fail
- Wait 4s, retry -> fail
- Wait 8s, retry -> success

Used by: AWS SDKs (built-in), Kinesis producers

### Step Functions Retry/Catch
Built-in error handling in state machine definition:
```json
{
  "Retry": [{
    "ErrorEquals": ["States.TaskFailed"],
    "IntervalSeconds": 3,
    "MaxAttempts": 3,
    "BackoffRate": 2.0
  }],
  "Catch": [{
    "ErrorEquals": ["States.ALL"],
    "Next": "HandleError"
  }]
}
```

---

## Throttling Patterns

| Service | Throttle Indicator | Fix |
|---------|-------------------|-----|
| **DynamoDB** | ProvisionedThroughputExceededException | Increase WCU/RCU, Auto-Scaling, On-Demand |
| **Kinesis** | ProvisionedThroughputExceeded | Split shards, On-Demand mode |
| **Lambda** | Throttle metric in CloudWatch | Increase reserved concurrency |
| **KMS** | ThrottlingException | Request quota increase |
| **API Gateway** | 429 Too Many Requests | Usage plans, throttling settings |

---

## Idempotency

- Ensure processing the same message twice produces the same result
- Critical for at-least-once delivery systems (SQS, Kinesis)
- Techniques: deduplication IDs, conditional writes, upsert patterns

---

## Exam Gotchas

- **DLQ** = answer for "handle failed messages without blocking the pipeline"
- **Exponential backoff** = answer for "handle throttling"
- Step Functions has built-in Retry/Catch - no custom code needed
- "Orchestration within code is an anti-pattern" - use Step Functions
- Lambda DLQ captures failed async invocations only (not sync)
- SQS visibility timeout must be longer than Lambda processing time
