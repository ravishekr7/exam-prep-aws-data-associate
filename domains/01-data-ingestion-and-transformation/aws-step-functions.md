# AWS Step Functions

Serverless state machine orchestrator for building workflows.

---

## State Types

| State | Purpose |
|-------|---------|
| **Task** | Do work (invoke Lambda, run Glue Job, call API) |
| **Choice** | Branching logic (if/else) |
| **Map** | Iterate over a list (parallel processing) |
| **Wait** | Pause for X seconds or until a timestamp |
| **Parallel** | Run multiple branches concurrently |
| **Pass** | Pass input to output (useful for transformations) |
| **Succeed/Fail** | End the execution |

---

## Workflow Types

| Feature | Standard | Express |
|---------|----------|---------|
| **Duration** | Up to 1 year | Max 5 minutes |
| **Execution** | Exactly-once | At-least-once |
| **Throughput** | Lower | High throughput (100K+/sec) |
| **Cost** | Per state transition | Per execution + duration |
| **Use Case** | Long-running ETL, human approval | IoT, high-volume event processing |

---

## Express Workflow Sub-Types

Express Workflows have two sub-types that the exam distinguishes. Both share the same limits (5-minute max, at-least-once, CloudWatch Logs only), but they differ in how the caller interacts with them:

| Sub-Type | Behaviour | Invocation API | Use Case |
|----------|-----------|----------------|----------|
| **Synchronous Express** | Caller **waits** and receives the execution result directly when it completes | `StartSyncExecution` — blocks until done, returns result in the API response | Real-time APIs, request/response pipelines where the caller needs the output immediately |
| **Asynchronous Express** | Fire-and-forget — execution starts immediately but the caller receives **no result** | `StartExecution` — returns immediately with an execution ARN; result goes to CloudWatch Logs | High-volume event ingestion, IoT pipelines, when the caller does not need the workflow's output |

### Key Distinctions
- **Synchronous Express** is the only Express type where the API response contains the workflow's final output
- **Asynchronous Express** is better for throughput-intensive scenarios where you just want to kick off work
- Both sub-types: max 5 minutes, at-least-once execution semantics, execution history only in CloudWatch Logs (not visible in the Step Functions console the way Standard executions are)

**Exam pattern:** "A REST API endpoint needs to trigger a workflow and return the result to the client in real time with low overhead" → **Synchronous Express Workflow** (`StartSyncExecution`).

---

## Integration Patterns

- **Request-Response:** Call service, get immediate response (default)
- **Run a Job (.sync):** Wait for the job to complete (Glue, EMR, ECS)
- **Wait for Callback (.waitForTaskToken):** Pause until external system calls back

---

## Input/Output Processing: InputPath, ResultPath, OutputPath

> **This is heavily tested.** Every Task state receives an input JSON and produces an output JSON. These three fields control exactly what flows into the task, what happens to the result, and what gets passed to the next state.

### Why It Matters
By default, a Task state's result **replaces** the entire input. If you don't control this, your original pipeline data is lost after the first task that returns anything.

---

### InputPath
Filters which part of the **incoming state input** is sent *to* the task resource. The task only sees the filtered portion, not the whole input.

- **Default:** `$` (entire input is sent to the task)
- **Example:** `"InputPath": "$.order"` — Lambda only receives the `order` field, not the whole state input object

---

### ResultPath
Controls **where the task's result is written** in the state's output. This is the most important of the three fields.

| ResultPath value | Effect |
|-----------------|--------|
| `"$"` (default) | Task result **replaces** the entire input. Original data is **lost**. |
| `"$.taskResult"` | Task result is **added** to the original input under the key `taskResult`. Original input is preserved. |
| `null` | Task result is **discarded entirely**. Original input passes through unchanged to the next state. |

> **Exam trap:** Not setting ResultPath (which defaults to `"$"`) means your Lambda's return value completely replaces the state input — your upstream pipeline data disappears. Almost always use a named sub-key.

- `"ResultPath": null` is useful for "side-effect" tasks like sending an SNS notification, where you want the workflow to invoke something but don't care about its output and don't want it to overwrite your data.

---

### OutputPath
Filters the **final state output** before it is passed to the next state. Applied *after* ResultPath. Use it to trim large outputs and pass only what the next state needs.

- **Default:** `$` (entire output is passed)
- **Example:** `"OutputPath": "$.validationResult"` — only the `validationResult` portion of the merged output is passed forward

---

### Worked Example

```json
// Scenario: Lambda validates an order. We want to keep the original order
// data AND add the validation result, then pass both to the next state.
{
  "Type": "Task",
  "Resource": "arn:aws:lambda:us-east-1:123456789012:function:ValidateOrder",
  "InputPath": "$.order",
  "ResultPath": "$.validationResult",
  "OutputPath": "$",
  "Next": "ProcessOrder"
}

// Input to this state:
// { "order": { "id": "123", "amount": 99.99 }, "metadata": { "source": "web" } }

// Lambda receives (due to InputPath): { "id": "123", "amount": 99.99 }
// Lambda returns: { "valid": true, "score": 0.98 }

// Output passed to ProcessOrder (due to ResultPath + OutputPath):
// {
//   "order": { "id": "123", "amount": 99.99 },
//   "metadata": { "source": "web" },
//   "validationResult": { "valid": true, "score": 0.98 }
// }
```

The original `order` and `metadata` are preserved. The Lambda's result is added alongside them.

---

## Error Handling — Retry and Catch (Full Syntax)

> **One of the most tested areas in DEA-C01.** Step Functions has built-in retry and catch at the state level — no custom Lambda error-handling code needed.

### Full Annotated Example

```json
"TransformData": {
  "Type": "Task",
  "Resource": "arn:aws:states:::glue:startJobRun.sync",
  "Parameters": {
    "JobName": "my-glue-etl-job"
  },
  "Retry": [
    {
      "ErrorEquals": ["Glue.AWSGlueException", "States.TaskFailed"],
      "IntervalSeconds": 30,
      "MaxAttempts": 3,
      "BackoffRate": 2.0,
      "JitterStrategy": "FULL"
    },
    {
      "ErrorEquals": ["States.ALL"],
      "IntervalSeconds": 60,
      "MaxAttempts": 1,
      "BackoffRate": 1.0
    }
  ],
  "Catch": [
    {
      "ErrorEquals": ["States.ALL"],
      "Next": "NotifyFailure",
      "ResultPath": "$.errorInfo"
    }
  ],
  "Next": "ValidateOutput"
}
```

### Field-by-Field Explanation

| Field | Meaning |
|-------|---------|
| `ErrorEquals` | List of error names to match. Step Functions evaluates Retry entries in order — first match wins. Put specific errors before `States.ALL`. |
| `IntervalSeconds` | Wait time in seconds before the **first** retry attempt. |
| `MaxAttempts` | Total number of retry attempts. After exhausting all retries, Step Functions evaluates the `Catch` block. |
| `BackoffRate` | Multiplier applied to the wait time after each retry. `BackoffRate: 2.0` with `IntervalSeconds: 30` → waits 30s, then 60s, then 120s (exponential backoff). |
| `JitterStrategy: "FULL"` | Adds random jitter to each wait interval. Prevents **thundering herd** — if 100 executions all fail at the same time, without jitter they all retry simultaneously and overwhelm the downstream service. `"FULL"` randomises each wait within the backoff window. |
| `Catch.ResultPath` | Where to write the error object in the state output before routing to the error state. `"$.errorInfo"` **preserves the original input** and appends the error details. `"$"` would replace everything with just the error — your pipeline context is lost. |

### How Retry and Catch Interact

```
State executes → fails
      ↓
Does a Retry rule match?
  YES → wait (IntervalSeconds × BackoffRate^attempt + jitter) → retry
  NO  → fall through to Catch immediately

After MaxAttempts exhausted on all Retry rules:
      ↓
Does a Catch rule match?
  YES → route to the specified Next state, write error to ResultPath
  NO  → execution FAILS (propagates the error up)
```

> **Rule: Retry is always evaluated before Catch.** Only after all retry attempts are exhausted does Step Functions check Catch.

### Exam Patterns
- "Workflow must retry a Glue job 3 times with increasing wait periods before alerting" → `Retry` with `BackoffRate > 1` and `MaxAttempts: 3`
- "Catch all errors and route to a notification Lambda while preserving the original input" → `Catch` with `ResultPath: "$.error"` (not `"$"`)
- "Prevent multiple concurrent failed executions from hammering a downstream API during retry" → `JitterStrategy: "FULL"`
- "Retry transient Lambda throttling errors but immediately fail on data validation errors" → Two separate Retry entries: specific error with `MaxAttempts: 3`, then a custom error with `MaxAttempts: 0`

---

## Distributed Map State

The **Distributed Map** state is a newer, high-scale variant of the regular Map state, designed for processing large datasets directly from S3. It is fundamentally different from the inline Map state.

### Inline Map vs Distributed Map

| Feature | Inline Map | Distributed Map |
|---------|-----------|-----------------|
| **Input source** | An array **in the state input** (in-memory JSON) | S3 objects, S3 inventory manifest, or a JSON array stored in S3 |
| **Scale** | Hundreds of iterations (bounded by input JSON size) | **Millions of iterations** (up to 10,000 concurrent child executions) |
| **Child execution type** | Child states run within the parent Standard execution | Launches **child Express Workflow executions** (for throughput and cost) |
| **Results aggregation** | Results returned inline in parent execution output | Results written to **S3** (too large for inline return) |
| **Use case** | Small parallel loops within a workflow (e.g., fan-out to 5 APIs) | Large-scale parallel S3 file processing |

### How It Works

```
Distributed Map state
  ├── ItemReader: points to S3 bucket/prefix or inventory file
  │     → Step Functions reads the list of objects/items automatically
  ├── MaxConcurrency: e.g., 1000 (run 1000 child executions in parallel)
  ├── ItemProcessor: the sub-workflow applied to each item
  │     → Each child execution is a separate Express Workflow
  └── ResultWriter: S3 path where aggregated results are written
```

### Example Use Case

"Process 5 million log files stored in S3, run a Lambda transformation on each file in parallel, and aggregate the results."

With inline Map: impossible — you cannot fit 5 million file paths in a single JSON input.
With Distributed Map: point the ItemReader at the S3 prefix, set MaxConcurrency to 10,000, and Step Functions orchestrates the entire parallel execution automatically.

**Exam pattern:** "Use Step Functions to process a large number of S3 objects in parallel" → **Distributed Map state**, not regular Map. Regular Map is for small in-memory arrays.

---

## Common Data Engineering Pattern

```
S3 Event -> EventBridge -> Step Functions:
  1. Run Glue Crawler (update catalog)
  2. Run Glue Job (transform data)
  3. Choice: Success?
     - Yes -> SNS "Success" notification
     - No  -> Lambda (cleanup) -> SNS "Failure" notification
```

---

## Exam Gotchas

- Step Functions is the answer for "orchestrate multiple AWS services with error handling"
- Use **.sync** integration to wait for Glue/EMR jobs to complete before next step
- Standard workflows for long ETL; Express for high-volume event processing
- Built-in Retry and Catch blocks for error handling (no custom code needed)
- "Orchestration within code is an anti-pattern" - use Step Functions instead of coding retry logic in Lambda
- **`ResultPath: "$"` is a trap.** Setting `ResultPath` to `"$"` in a Task state or Catch block replaces the **entire** state input with the task output or error object — all upstream data is lost. Almost always use a named sub-key: `"ResultPath": "$.result"` or `"ResultPath": "$.error"` to preserve pipeline context alongside the new data.
- **Retry is evaluated BEFORE Catch.** Step Functions retries up to `MaxAttempts` first. Only after all retries are exhausted does it evaluate Catch. If no Catch rule matches after retries, the entire execution fails with the error.
- **waitForTaskToken pattern for human approval:** Append `.waitForTaskToken` to the Resource ARN (e.g., `arn:aws:states:::sqs:sendMessage.waitForTaskToken`). Step Functions pauses the execution and embeds a task token in the message. Your external system (human approval portal, another microservice) calls `SendTaskSuccess` or `SendTaskFailure` with that token to resume. The execution can wait up to 1 year (Standard Workflow). This is the correct pattern for "a pipeline needs a human to approve a step before proceeding."
- **`States.Timeout` is not retried by default.** If a Task times out (exceeds `TimeoutSeconds` or `HeartbeatSeconds`), it throws `States.Timeout`. This error is **not** covered by a `States.TaskFailed` Retry rule — you must explicitly add `States.Timeout` to your `ErrorEquals` list if you want timeouts retried.
- **Standard Workflow billing is per state transition.** A Map state iterating over 1,000 items through 5 states each = 5,000 state transitions billed. For high-volume loops or iterating over large datasets, use Express Workflows (billed per execution + duration) to dramatically reduce cost.
