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

## Step Functions Error Handling — Full Syntax

The following shows a complete, production-ready Retry + Catch configuration for a Lambda task state. This exact pattern is tested on the exam.

```json
{
  "ProcessData": {
    "Type": "Task",
    "Resource": "arn:aws:lambda:us-east-1:123456789012:function:ProcessDataFunction",
    "Retry": [
      {
        "ErrorEquals": ["Lambda.ServiceException", "Lambda.TooManyRequestsException"],
        "IntervalSeconds": 2,
        "MaxAttempts": 3,
        "BackoffRate": 2.0,
        "JitterStrategy": "FULL"
      }
    ],
    "Catch": [
      {
        "ErrorEquals": ["States.ALL"],
        "Next": "ErrorNotificationState",
        "ResultPath": "$.error"
      }
    ],
    "Next": "SuccessState"
  }
}
```

### Field Explanations

| Field | Meaning |
|-------|---------|
| `ErrorEquals` | List of error types to match. `Lambda.ServiceException` = Lambda infrastructure error. `Lambda.TooManyRequestsException` = Lambda concurrency throttling. `States.ALL` = catch any error not already caught. |
| `IntervalSeconds` | Initial wait time in seconds before the first retry. Here: 2 seconds. |
| `BackoffRate` | Multiplier applied to wait time after each retry. Rate of 2.0 = exponential backoff (2s → 4s → 8s). |
| `MaxAttempts` | Maximum number of retry attempts. After 3 attempts, Step Functions stops retrying and moves to the Catch block. |
| `JitterStrategy` | `"FULL"` adds a random jitter (0 to IntervalSeconds × BackoffRate^attempt) to the wait time. Prevents thundering herd — all retrying tasks firing simultaneously after an outage. |
| `ResultPath` | Where to store the error info in the state input. `"$.error"` **adds** the error object to the existing input. Using `"$"` would **overwrite** the entire input with the error — almost always wrong. |

### ResultPath Best Practice
```json
// CORRECT: preserve original input, add error details
"ResultPath": "$.error"
// State input becomes: { "original_field": "value", "error": { "Cause": "...", "Error": "..." } }

// WRONG: overwrites entire input with error object
"ResultPath": "$"
// State input becomes: { "Cause": "...", "Error": "..." }  ← original data lost!
```

### Retry for Specific Error Types
```json
"Retry": [
  {
    "ErrorEquals": ["Lambda.TooManyRequestsException"],
    "IntervalSeconds": 1,
    "MaxAttempts": 5,
    "BackoffRate": 2.0,
    "JitterStrategy": "FULL"
  },
  {
    "ErrorEquals": ["CustomError.DataValidationFailed"],
    "MaxAttempts": 0
  }
]
```
Order matters: Step Functions matches the first applicable error type. Put specific errors before `States.ALL`.

---

## Kinesis + Lambda Error Handling

When Lambda processes records from a Kinesis Data Stream, failed batches can block the shard indefinitely (Lambda keeps retrying the same batch). These settings prevent that:

### Key Event Source Mapping Settings

| Setting | Purpose | Recommendation |
|---------|---------|---------------|
| `bisectBatchOnFunctionError: true` | If a batch fails, split it in half and retry each half independently. Narrows down which record is poisonous in O(log n) steps instead of retrying the full batch repeatedly. | Enable in production |
| `maximumRetryAttempts` | Limit the number of times Lambda retries a failing batch. Default: unlimited (dangerous — can block a shard for hours/days). | Set to 3-5 for most pipelines |
| `maximumRecordAgeInSeconds` | Skip records older than N seconds. If a poison-pill record has been blocking the shard for hours, this setting eventually skips it and moves on. | Set to 3600 (1 hour) or per SLA |
| `destinationConfig.onFailure` | Send the failed records (and error metadata) to an SQS DLQ or SNS topic instead of discarding them silently. | Always configure a DLQ |

### Complete Configuration Example
```json
{
  "EventSourceArn": "arn:aws:kinesis:us-east-1:123456789:stream/orders",
  "FunctionName": "process-orders",
  "StartingPosition": "TRIM_HORIZON",
  "BatchSize": 100,
  "BisectBatchOnFunctionError": true,
  "MaximumRetryAttempts": 3,
  "MaximumRecordAgeInSeconds": 3600,
  "DestinationConfig": {
    "OnFailure": {
      "Destination": "arn:aws:sqs:us-east-1:123456789:orders-dlq"
    }
  }
}
```

### IteratorAgeMilliseconds Alert
This CloudWatch metric measures how far behind the stream consumer is (age of the oldest record currently being processed). If this metric is **trending upward** over time, the consumer is falling behind:

```
CloudWatch Alarm:
Metric: IteratorAgeMilliseconds
Statistic: Maximum
Threshold: 60000 (60 seconds behind)
Action: SNS → Scale Lambda concurrency OR increase shard count
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

## SQS Dead Letter Queue Pattern

### maxReceiveCount
The number of times a message can be received (dequeued) from the source queue before being moved to the DLQ. Each time Lambda (or any consumer) picks up a message and fails to process it, the receive count increments.

```
Source Queue (maxReceiveCount = 3):
  Attempt 1 → Lambda fails → message becomes visible again
  Attempt 2 → Lambda fails → message becomes visible again
  Attempt 3 → Lambda fails → message moved to DLQ
```

### Complete DLQ Monitoring Pattern
```
1. Source SQS Queue (maxReceiveCount = 3)
          ↓ (after 3 failures)
2. Dead Letter Queue (SQS)
          ↓
3. CloudWatch Alarm on DLQ metric:
   ApproximateNumberOfMessagesVisible > 0
          ↓
4. SNS notification → ops team email/Slack
          ↓
5. Lambda (triggered by DLQ) to inspect failed messages, log details, and optionally:
   - Fix and replay to source queue
   - Archive to S3 for investigation
   - Alert with specific failure reason
```

### Visibility Timeout Critical Rule
```
SQS Visibility Timeout MUST be >= Lambda function timeout

Example:
  Lambda timeout: 15 minutes
  SQS visibility timeout: must be >= 15 minutes (set to 20 minutes for safety)

Why: If Lambda takes 14 minutes to process a message and the visibility timeout
is only 10 minutes, the message becomes visible again after 10 minutes and gets
processed a SECOND TIME by another Lambda instance — duplicate processing!
```

### SQS FIFO with DLQ
For FIFO queues, failed messages block the queue (messages are processed in order, so a failed message at position 3 blocks messages 4, 5, 6...). Always configure a DLQ for FIFO queues to prevent queue blocking.

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
