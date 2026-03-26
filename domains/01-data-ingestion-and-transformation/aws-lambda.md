# AWS Lambda

Serverless compute service that runs code in response to events.

---

## Key Specs

| Limit | Value |
|-------|-------|
| **Timeout** | Max 15 minutes |
| **Memory** | 128 MB to 10 GB |
| **/tmp Storage** | 512 MB default, configurable up to 10,240 MB (10 GB) — increase via console/CLI |
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

## Concurrency Deep Dive

Lambda concurrency has three distinct modes. The exam tests all three and their throttle behaviour.

| Mode | What It Does | Cost | Use Case |
|------|-------------|------|----------|
| **Unreserved** | Default pool shared across all functions in the region | No extra charge | Functions with no specific SLA |
| **Reserved** | Guarantees N concurrent instances for this function, subtracts from the regional pool | No extra charge | Critical functions that must never be starved by other functions |
| **Provisioned** | Pre-initialises N execution environments — eliminates cold starts entirely | Additional charge (per provisioned concurrency-hour) | Latency-sensitive APIs, functions called by users with strict p99 requirements |

### Throttle Behaviour (Exam-Critical)

- When a function **exceeds its reserved concurrency limit** → `HTTP 429 TooManyRequestsException` returned immediately to the caller
- When the **regional concurrency limit** (default 1,000) is hit → all unreserved functions are throttled
- **Synchronous invocations** (API Gateway → Lambda): throttled request gets a 429 — the caller must handle the retry
- **Asynchronous invocations** (S3 event → Lambda): throttled request is retried automatically for up to **6 hours** with exponential backoff before being sent to the DLQ or Destination

### Concurrency Calculation the Exam Uses

```
concurrent_executions = invocations_per_second × avg_duration_seconds

Example: 500 requests/sec, each takes 2 seconds average
= 500 × 2 = 1,000 concurrent executions needed
→ This hits the default regional limit exactly
→ Fix: request a limit increase OR optimise the function to run faster
```

### Exam Patterns
- "Lambda function is being throttled even though the regional limit is not hit" → function has **reserved concurrency set too low**
- "A critical Lambda must never be starved by other functions in the account" → set **reserved concurrency**
- "Java Lambda has high cold start latency on first invocation after idle period" → enable **Provisioned Concurrency**

---

## Event Source Mapping (ESM) — Kinesis and DynamoDB Streams

> **One of the most tested Lambda topics in DEA-C01.** The exam writes scenario questions around exactly these settings — know each one and what happens if you leave it at default.

When Lambda reads from Kinesis Data Streams or DynamoDB Streams via Event Source Mapping, it polls the stream and invokes your function with a batch of records. By default, a single failing record can block the entire shard indefinitely.

### Critical ESM Settings

| Setting | Default | What It Controls | Exam Relevance |
|---------|---------|-----------------|----------------|
| `bisectBatchOnFunctionError` | `false` | On failure, splits batch in half and retries each half separately — isolates bad records in O(log n) retries instead of retrying the whole batch forever | **HIGH** |
| `maximumRetryAttempts` | `-1` (unlimited) | Max number of times to retry a failing batch before skipping it and moving on | **HIGH** |
| `maximumRecordAgeInSeconds` | `-1` (unlimited) | Skip records older than N seconds — prevents an ancient stuck record from blocking the shard indefinitely | **HIGH** |
| `destinationConfig.onFailure` | None | SQS queue or SNS topic to receive records that exhausted all retries — without this, failed records are silently discarded | **HIGH** |
| `startingPosition` | (required) | `TRIM_HORIZON` (process all records from oldest) vs `LATEST` (only new records going forward) | MEDIUM |
| `batchSize` | 100 | Number of records per Lambda invocation | MEDIUM |
| `parallelizationFactor` | 1 | Number of concurrent Lambda invocations **per shard** (up to 10) — increases throughput within a single shard | MEDIUM |

### The Poison-Pill Problem and Solution

Without tuning, a single malformed record blocks an entire Kinesis shard indefinitely:

```
Default behaviour (dangerous):
  Bad record arrives → Lambda fails → retries same batch → fails again
  → repeats until record expires (up to 7 days with extended retention)
  → shard is effectively blocked for days

Fixed configuration:
  bisectBatchOnFunctionError: true     # splits batch in half on failure
  maximumRetryAttempts: 3              # give up after 3 attempts per sub-batch
  maximumRecordAgeInSeconds: 3600      # skip anything older than 1 hour regardless
  destinationConfig:
    onFailure:
      destination: arn:aws:sqs:us-east-1:123456789:poison-pill-dlq
      # captures bad records for inspection instead of silently dropping them
```

### Lambda Filter Policies for ESM

Filter which stream records trigger Lambda invocations at the source. Records not matching the filter are skipped entirely before reaching Lambda — reduces invocations and cost.

```json
// Only trigger Lambda for DynamoDB Stream INSERT events (not MODIFY or REMOVE)
{
  "Filters": [
    {
      "Pattern": {
        "eventName": ["INSERT"]
      }
    }
  ]
}
```

```json
// Only trigger Lambda when a Kinesis record contains a specific field value
{
  "Filters": [
    {
      "Pattern": {
        "data": {
          "eventType": ["PAYMENT_FAILED"]
        }
      }
    }
  ]
}
```

**Exam pattern:** "Reduce Lambda invocation costs for a DynamoDB Stream that has high write volume but only INSERT events are relevant to the downstream process" → add an ESM filter policy for `eventName = INSERT`.

### parallelizationFactor

- Default: 1 Lambda invocation per shard at a time (sequential within a shard — one batch finishes before the next starts)
- Set to 2–10 to process multiple batches from the same shard concurrently
- Useful when a high-throughput shard produces more records than one Lambda invocation can consume in time
- **Tradeoff:** Records with the same partition key may be processed out of order if factor > 1 — design your function idempotently

---

## Lambda Destinations vs DLQ

Lambda has two mechanisms for routing the outcomes of failed (and successful) invocations. The exam tests when to use each.

### DLQ (Dead Letter Queue)
- Applies to **asynchronous invocations only** (S3 events, SNS, EventBridge)
- After Lambda's 2 built-in retries are exhausted: sends the **original event payload** to SQS or SNS
- Only handles **failures** — no success routing
- Does **not** work with Event Source Mapping (Kinesis, DynamoDB Streams, SQS)
- Older feature, simpler configuration

### Lambda Destinations (Newer, More Flexible)
- Applies to **asynchronous invocations** AND **Event Source Mapping** (Kinesis, SQS, DynamoDB Streams)
- Routes to SQS, SNS, EventBridge, or another Lambda function
- Handles **both** `onSuccess` and `onFailure` routing separately
- For ESM `onFailure`: destination receives a **rich metadata envelope** — failed record payload, function ARN, error details, shard ID, sequence numbers — much more context than a bare DLQ payload

```
DLQ:          async failure only → SQS/SNS
              (original event payload only, no metadata)

Destinations: async failure → SQS/SNS/EventBridge/Lambda
              (rich envelope: error details + original event + execution context)
              async success → SQS/SNS/EventBridge/Lambda
              (result + original event)
```

### Decision Rule for the Exam

| Scenario | Answer |
|----------|--------|
| Need to route **both successes and failures** for async invocations | Destinations |
| Need failure routing for **ESM (Kinesis/DynamoDB Streams)** | Destinations (`destinationConfig.onFailure`) — DLQ does not support ESM |
| Simple async failure capture, legacy system, SQS only | DLQ is fine |
| "Capture failed Kinesis records to SQS for inspection without blocking the stream" | ESM + `destinationConfig.onFailure` (this is a Destinations config, not a traditional DLQ) |

---

## Lambda SnapStart and Container Images

### Lambda SnapStart (Java Only)

SnapStart dramatically reduces cold start latency for Java functions without code changes.

**How it works:**
1. When you publish a new Lambda version, AWS initialises the function once (runs `init` code — class loading, dependency injection, etc.)
2. Takes a memory and disk **snapshot** of the fully initialised execution environment
3. Caches the snapshot
4. Subsequent cold starts restore from the cached snapshot instead of re-running init — cold start drops from 3–10 seconds to under 1 second

**Key constraints:**
- Available for **Java 11 and later** runtimes only
- Requires Lambda **versions or aliases** — not available for `$LATEST`
- No extra charge for SnapStart itself — standard Lambda rates apply
- Snapshot is taken at publish time, so if your init code has time-dependent state (e.g., reads current timestamp on startup), behaviour may differ slightly at restore time

**Exam pattern:** "Java Lambda function has 3-second cold start latency causing p99 API latency issues — how to fix with the least code change?" → Enable **Lambda SnapStart** (zero code changes). Provisioned Concurrency also eliminates cold starts but costs extra per pre-initialised environment.

### Lambda Container Images

| Deployment Type | Max Size | Notes |
|----------------|----------|-------|
| Zip package | 50 MB (zipped), 250 MB (unzipped) | Standard, fastest cold start |
| Container image | **10 GB** | Stored in ECR; use for large ML models, complex system dependencies, or custom runtimes |

- Container images must include the **Lambda Runtime Interface Client (RIC)** — this is what connects your container to the Lambda service
- Same Lambda programming model (handler function, event/context object) — just a different packaging format
- Cold starts are slightly longer for containers vs zip for the same code size, but the 10 GB limit opens up use cases that zip cannot serve
- **Exam pattern:** "Lambda function requires a 2 GB machine learning model file at runtime — how to package it?" → Container image (zip unzipped limit is 250 MB, too small for a 2 GB model)

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
- **Reserved concurrency = 0 is a valid way to disable a Lambda function.** Setting reserved concurrency to 0 means all invocations are immediately throttled (zero concurrent executions allowed). Use this to disable a function temporarily without deleting it — e.g., disable a Kinesis consumer while investigating a data quality incident.
- **Provisioned concurrency requires a published version or alias — NOT `$LATEST`:** You cannot attach provisioned concurrency to the unpublished `$LATEST` version. Publish a numbered version first (or create an alias pointing to it), then configure provisioned concurrency on that version/alias.
- **Lambda in VPC cold start is no longer slow:** The old 10–30 second VPC cold start was caused by ENI creation at invocation time. AWS solved this with Hyperplane ENIs (pre-created, shared). VPC Lambda cold starts are now comparable to non-VPC. Exam questions that describe "Lambda in VPC is inherently slow" are using outdated information — this is a deliberate distractor.
- **Lambda /tmp is NOT shared across concurrent invocations.** Each Lambda execution environment has its own isolated `/tmp` directory. Data written by one invocation is invisible to another concurrent invocation. `/tmp` IS persistent across sequential invocations on the same warm execution environment — but you must not rely on this (the environment may be recycled at any time).
- **ESM failure behaviour differs between SQS and Kinesis:** For SQS ESM, Lambda deletes successfully processed messages and leaves failed ones in the queue (they reappear after the visibility timeout). For **Kinesis ESM**, Lambda blocks on the failing record by default — it does not skip ahead to newer records. This asymmetry is why `bisectBatchOnFunctionError` and `maximumRetryAttempts` are critical for Kinesis consumers but less critical for SQS consumers.
