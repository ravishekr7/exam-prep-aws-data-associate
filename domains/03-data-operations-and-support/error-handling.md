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

Ensure processing the same message twice produces the same result — critical for **at-least-once delivery** systems (SQS, Kinesis, SNS).

### Idempotency Implementation Patterns

**1. Conditional Write (DynamoDB)**
```python
# Only write if the item doesn't already exist
try:
    dynamodb.put_item(
        TableName='processed-events',
        Item={'event_id': {'S': event_id}, 'result': {'S': result}},
        ConditionExpression='attribute_not_exists(event_id)'
    )
except dynamodb.exceptions.ConditionalCheckFailedException:
    # Already processed — skip safely
    return
```

**2. SQS FIFO Deduplication**
- Set `MessageDeduplicationId` on each message (hash of message body)
- SQS FIFO deduplicates within a **5-minute window** — same ID = message silently dropped
- Use content-based deduplication: SQS hashes the message body automatically

**3. Upsert Pattern (INSERT OR REPLACE)**
```sql
-- Redshift / PostgreSQL: idempotent upsert
INSERT INTO orders (order_id, amount, status)
VALUES (%(order_id)s, %(amount)s, %(status)s)
ON CONFLICT (order_id) DO UPDATE SET
  amount = EXCLUDED.amount,
  status = EXCLUDED.status;
```

**4. Idempotency Key in HTTP/API Requests**
- Pass a client-generated UUID as `Idempotency-Key` header
- Server stores key + response; returns same response for repeated requests with same key
- AWS Step Functions `startExecution` supports `name` parameter as idempotency key

### Idempotency Record Cleanup

- Store processed IDs with a TTL (DynamoDB TTL attribute)
- Set TTL = message retention period of the source queue
- After TTL, the record is deleted — duplicate protection no longer needed for old messages

---

## Circuit Breaker Pattern

Prevents cascading failures by stopping calls to a consistently failing downstream service.

### States

```
CLOSED → (failures < threshold) → normal operation, all calls pass through
   ↓ (failures exceed threshold)
OPEN → (all calls fail immediately, no downstream calls) → wait cooldown period
   ↓ (after cooldown)
HALF-OPEN → (allow one test call through)
   ↓ success              ↓ failure
CLOSED (reset)         OPEN (back to waiting)
```

### Implementation with DynamoDB

```python
# DynamoDB item tracks circuit state per service
{
  "service_name": "payment-api",
  "state": "OPEN",           # CLOSED, OPEN, HALF-OPEN
  "failure_count": 7,
  "last_failure_time": 1712000000,
  "cooldown_seconds": 60
}

def call_with_circuit_breaker(service_name, call_fn):
    state = get_circuit_state(service_name)  # read from DynamoDB
    if state == "OPEN":
        if time.time() - state.last_failure_time > state.cooldown_seconds:
            set_state(service_name, "HALF-OPEN")
        else:
            raise CircuitOpenException("Service unavailable")
    try:
        result = call_fn()
        reset_circuit(service_name)  # success → CLOSED
        return result
    except Exception as e:
        increment_failure(service_name)  # failure → OPEN if threshold exceeded
        raise
```

> **Exam tip:** Circuit breaker = DynamoDB state store + Lambda logic. Prevents a slow downstream system from causing Lambda timeouts and exhausting concurrency across the whole account.

---

## AWS X-Ray (Distributed Tracing)

Track requests as they flow across Lambda, API Gateway, SQS, DynamoDB, and other services.

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Trace** | End-to-end record of a single request across all services |
| **Segment** | One service's contribution to a trace (e.g., Lambda execution) |
| **Subsegment** | Sub-operation within a segment (e.g., DynamoDB call inside Lambda) |
| **Annotation** | Indexed key-value pair — use for filtering traces (e.g., `customer_id`) |
| **Metadata** | Non-indexed data attached to a segment (for debugging, not filtering) |
| **Sampling** | X-Ray samples a percentage of requests — not all traces are recorded by default |

### Enable X-Ray on Lambda

```python
# Install: pip install aws-xray-sdk
from aws_xray_sdk.core import xray_recorder, patch_all
patch_all()  # auto-instruments boto3, requests, etc.

@xray_recorder.capture('process_order')
def handler(event, context):
    with xray_recorder.in_subsegment('validate'):
        validate_order(event)
    with xray_recorder.in_subsegment('write_to_dynamodb'):
        write_order(event)
```

### X-Ray Service Map

- Visual map of all services involved in a request flow
- Shows latency, error rate, and fault rate per node
- Helps identify bottlenecks: which service is slow or throwing errors?

### Correlation IDs

Pass a unique `correlation_id` through all services in a pipeline for end-to-end traceability:

```python
# Lambda receives event with correlation_id
correlation_id = event.get('correlation_id', str(uuid.uuid4()))

# Add to X-Ray annotation for filtering
xray_recorder.put_annotation('correlation_id', correlation_id)

# Pass downstream (SQS message, Step Functions input, etc.)
sqs.send_message(
    QueueUrl=queue_url,
    MessageBody=json.dumps({**payload, 'correlation_id': correlation_id})
)
```

---

## Service-Specific Error Reference

### Lambda

| Error | Cause | Fix |
|-------|-------|-----|
| `Lambda.TooManyRequestsException` | Concurrency limit hit | Increase reserved concurrency or use SQS buffer |
| `Lambda.ServiceException` | Lambda infrastructure issue | Retry with backoff (transient) |
| `Lambda.AWSLambdaException` | Function execution error | Fix function code |
| `Lambda.SdkClientException` | SDK/network issue | Retry with backoff |
| `Task timed out` | Function exceeded timeout setting | Optimize code or increase timeout |

### Glue

| Error | Cause | Fix |
|-------|-------|-----|
| `OUT_OF_MEMORY` | Job needs more memory | Upgrade to G.2X workers |
| `TIMEOUT` | Job ran longer than timeout | Increase `--timeout` parameter |
| `RESOURCE_NOT_FOUND` | Table/database missing in catalog | Run crawler first or check table name |
| Job stuck in RUNNING | DPU not releasing | Check for infinite loops; set job timeout |

### Kinesis

| Error | Cause | Fix |
|-------|-------|-----|
| `ProvisionedThroughputExceededException` | Shard limit hit | Reshard or switch to On-Demand |
| `ExpiredIteratorException` | Iterator not used within 5 minutes | Get a new shard iterator |
| `ResourceNotFoundException` | Stream doesn't exist | Check stream name and region |

### Athena

| Error | Cause | Fix |
|-------|-------|-----|
| `QUERY_QUEUE_TIMEOUT` | Workgroup concurrency limit | Reduce concurrent queries or increase limit |
| `HIVE_EXCEEDED_PARTITION_LIMIT` | Too many partitions (> 20K default) | Use partition projection |
| `S3 Access Denied` | IAM/bucket policy issue | Grant `s3:GetObject` on data prefix |
| `SYNTAX_ERROR` | Invalid SQL | Check engine version (v2 vs v3 differences) |

---

## Exam Gotchas

- **DLQ** = answer for "handle failed messages without blocking the pipeline"
- **Exponential backoff** = answer for "handle throttling"
- **Step Functions Retry/Catch** = built-in, no custom retry code needed
- **"Orchestration within code is an anti-pattern"** → use Step Functions
- **Lambda DLQ captures failed async invocations only** — not synchronous invocations
- **SQS visibility timeout must be >= Lambda timeout** — or messages get processed twice
- **`bisectBatchOnFunctionError`** = isolates the bad record in a Kinesis/DynamoDB Streams batch in O(log n) retries
- **`maximumRecordAgeInSeconds`** = the escape valve for poison-pill records blocking a Kinesis shard forever
- **`ResultPath: "$.error"` in Catch** — always use this to preserve original input alongside the error; `"$"` overwrites input (wrong)
- **`JitterStrategy: "FULL"`** — prevents thundering herd after an outage when all retrying tasks fire at the same instant
- **Idempotency** — SQS standard queues are at-least-once; always design Lambda consumers to be idempotent
- **SQS FIFO deduplication window = 5 minutes** — duplicate messages within 5 min are silently dropped
- **X-Ray** = distributed tracing across services; annotation = indexed (filterable), metadata = non-indexed (debugging only)
- **Circuit breaker** = use DynamoDB to track state (CLOSED/OPEN/HALF-OPEN); prevents cascading failures from slow downstream services
