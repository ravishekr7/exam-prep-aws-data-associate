# Orchestration

Coordinating workflows across multiple AWS services.

---

## Service Comparison

| Feature | Step Functions | MWAA (Airflow) | Glue Workflows | EventBridge |
|---------|---------------|----------------|----------------|-------------|
| **Type** | State machine | DAG orchestrator | ETL workflow | Event router |
| **Serverless** | Yes | No (managed env) | Yes | Yes |
| **Definition** | JSON/YAML (ASL) | Python (DAGs) | Visual/API | Event rules |
| **Idle cost** | None (pay per transition) | Yes (env always running) | None | None |
| **Best for** | AWS service orchestration | Complex dependencies, backfilling | Glue-only workflows | Event-driven triggers |
| **Error handling** | Native Retry/Catch per state | Task-level retries, callbacks | Job-level retries | DLQ on rule targets |
| **Max execution** | 1 year (Standard), 5 min (Express) | Unlimited (Airflow) | No hard limit | N/A |

---

## AWS Step Functions

### Workflow Types

| Type | Duration | Execution history | Cost model | Use case |
|------|----------|-------------------|------------|----------|
| **Standard** | Up to 1 year | Full audit trail in console | Per state transition | Long-running, auditable ETL |
| **Express** | Up to 5 minutes | CloudWatch Logs only | Per execution + duration | High-volume, short tasks |

> **Exam tip:** Standard = long-running + full history. Express = high-throughput + cheap (millions/day).

### State Types (Amazon States Language)

| State | Purpose |
|-------|---------|
| **Task** | Call an AWS service or Lambda |
| **Pass** | Pass input to output, optionally inject data |
| **Wait** | Pause for a fixed time or until timestamp |
| **Choice** | Branch based on conditions (if/else logic) |
| **Parallel** | Execute branches concurrently, wait for all |
| **Map** | Iterate over an array, run state machine per item |
| **Succeed** | End execution successfully |
| **Fail** | End execution with an error |

### Input/Output Path Processing

Step Functions processes JSON through a pipeline of filters per state:

```
InputPath  →  Parameters  →  Task execution  →  ResultSelector  →  ResultPath  →  OutputPath
```

| Field | What it does |
|-------|-------------|
| `InputPath` | Selects a portion of the state input to pass forward (`$` = all) |
| `Parameters` | Constructs a new JSON object as input to the task |
| `ResultSelector` | Picks fields from the raw task result |
| `ResultPath` | Where to put the task result in the state input (`null` = discard result) |
| `OutputPath` | What to pass to the next state |

```json
// Example: add task result to input without replacing it
"ResultPath": "$.taskResult"

// Example: discard result, pass original input unchanged
"ResultPath": null

// Example: only pass a specific field from the result
"OutputPath": "$.taskResult.jobId"
```

### Service Integration Patterns

| Pattern | Suffix | Behavior |
|---------|--------|---------|
| **Request/Response** | (none) | Fire and forget — Step Functions continues immediately |
| **Synchronous (sync)** | `.sync:2` | Step Functions waits for the job to complete |
| **Callback (waitForTaskToken)** | `.waitForTaskToken` | Step Functions pauses until external system sends `SendTaskSuccess`/`SendTaskFailure` |

```json
// Sync integration — wait for Glue job to finish
"Resource": "arn:aws:states:::glue:startJobRun.sync:2"

// Callback — pause until an external worker sends back a task token
"Resource": "arn:aws:states:::sqs:sendMessage.waitForTaskToken",
"Parameters": {
  "QueueUrl": "...",
  "MessageBody": {
    "TaskToken.$": "$$.Task.Token"
  }
}
```

> **Exam tip:** Use `.sync:2` for Glue/EMR/ECS jobs you need to wait on. Use `.waitForTaskToken` for human-approval steps or long-running external processes.

### Error Handling

```json
"Retry": [
  {
    "ErrorEquals": ["States.TaskFailed"],
    "IntervalSeconds": 5,
    "MaxAttempts": 3,
    "BackoffRate": 2.0
  }
],
"Catch": [
  {
    "ErrorEquals": ["States.ALL"],
    "Next": "HandleFailureState",
    "ResultPath": "$.error"
  }
]
```

- `BackoffRate`: Multiplier for wait between retries (2.0 = exponential backoff)
- `ResultPath` in Catch: Adds the error info to the state data instead of replacing it
- Common error classes: `States.TaskFailed`, `States.Timeout`, `States.ALL`, `Lambda.ServiceException`

### Map State (Parallel Processing)

```json
"Type": "Map",
"ItemsPath": "$.files",
"MaxConcurrency": 10,
"Iterator": { ... }
```

- Runs the `Iterator` state machine once per item in the array
- `MaxConcurrency`: Limits parallel iterations (0 = unlimited)
- Results collected into an array and passed to the next state

### Pricing

- Standard: $0.025 per 1,000 state transitions
- Express: $1.00 per 1M executions + $0.00001667 per GB-second
- Free tier: 4,000 state transitions/month (Standard)

---

## Amazon MWAA (Managed Workflows for Apache Airflow)

### Key Concepts

- **DAG (Directed Acyclic Graph):** Python file defining the workflow; stored in S3
- **Task:** A single unit of work (operator call)
- **Operator:** Pre-built task template (e.g., `GlueJobOperator`, `S3KeySensor`)
- **Scheduler:** Polls DAGs, triggers tasks when dependencies are met
- **Worker:** Executes the actual tasks

### Environment Configuration

| Setting | Options | Notes |
|---------|---------|-------|
| **Airflow version** | 2.x | Choose at environment creation |
| **Worker class** | `mw1.small`, `mw1.medium`, `mw1.large` | Determines CPU/memory per worker |
| **Min/Max workers** | 1–25 | Auto-scales within this range based on queue depth |
| **Scheduler instances** | 2 recommended | Provides HA for the scheduler |

> **MWAA is NOT serverless** — you pay for the environment even when no DAGs are running.

### Common Airflow Operators for AWS

| Operator | Purpose |
|----------|---------|
| `GlueJobOperator` | Trigger Glue ETL job |
| `GlueCrawlerOperator` | Run a Glue Crawler |
| `AthenaOperator` | Execute Athena SQL query |
| `EmrCreateJobFlowOperator` | Create EMR cluster |
| `EmrAddStepsOperator` | Add steps to EMR cluster |
| `EmrJobFlowSensor` | Wait for EMR cluster state |
| `S3KeySensor` | Wait for a file to appear in S3 |
| `RedshiftSQLOperator` | Run SQL on Redshift |
| `SageMakerTrainingOperator` | Trigger SageMaker training job |
| `LambdaInvokeFunctionOperator` | Invoke Lambda function |

### Backfilling

- **Backfill:** Re-run a DAG for historical date ranges
- Airflow's `start_date` and `schedule_interval` allow automatic backfilling when a DAG is enabled
- Run via CLI: `airflow dags backfill -s 2024-01-01 -e 2024-03-01 my_dag`
- Respects task dependencies — will re-run only failed/skipped tasks

> **Exam tip:** Backfilling is a unique MWAA/Airflow capability. If the question mentions "re-process historical data by date range," the answer is MWAA.

### DAG Best Practices

- Store DAGs in S3; MWAA syncs them automatically
- Use `catchup=False` to prevent automatic backfilling on new DAGs
- Use `on_failure_callback` to send alerts (SNS/Slack)
- Keep task logic out of the DAG file — use operators or external scripts

---

## Amazon EventBridge

### Core Concepts

- **Event Bus:** Receives events (default, custom, or partner)
- **Rule:** Matches events by pattern and routes to targets
- **Target:** Where matched events are sent (Lambda, Step Functions, SQS, SNS, Kinesis, etc.)
- **Scheduler:** Cron or rate-based triggers (replaces CloudWatch Events)

### Event Bus Types

| Type | Description |
|------|-------------|
| **Default** | Receives AWS service events automatically |
| **Custom** | For your own application events |
| **Partner** | Receives events from SaaS partners (Salesforce, Datadog, etc.) |

### Cross-Account Event Routing

- Create a **resource policy** on the target account's event bus
- Source account sends events to the target account's bus ARN
- Useful for centralized event processing across an AWS Organization

### Event Pattern Matching

```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": {
    "bucket": { "name": ["my-data-bucket"] },
    "object": { "key": [{ "prefix": "raw/" }] }
  }
}
```

- Supports prefix, suffix, wildcard (`*`), `exists`, `numeric` comparisons, `anything-but`
- Max event size: **256 KB**

### Input Transformation

Transform the event before sending to the target — extract only needed fields:

```json
"InputTransformer": {
  "InputPathsMap": {
    "bucket": "$.detail.bucket.name",
    "key": "$.detail.object.key"
  },
  "InputTemplate": "{\"bucket\": \"<bucket>\", \"key\": \"<key>\"}"
}
```

### Archive and Replay

- **Archive:** Store all (or filtered) events on a bus for a retention period
- **Replay:** Re-send archived events to the bus (replays as if they happened now)
- Use case: Replay events after fixing a bug in a consumer Lambda

### Dead-Letter Queue (DLQ) for Rules

- If EventBridge cannot deliver an event to a target, it retries with backoff
- After retries exhausted, event goes to an **SQS DLQ** attached to the target
- Configure DLQ per target in the rule

### EventBridge Scheduler

- Standalone scheduler (not event-bus-based) for one-time or recurring schedules
- Supports cron expressions and rate expressions
- Can invoke any AWS API action as a target (not just Lambda/Step Functions)

### Common Patterns

```
S3 Object Created → EventBridge Rule → Step Functions (start ETL workflow)
Schedule (cron) → EventBridge → Glue Job (nightly batch)
DynamoDB Stream → EventBridge Pipes → Lambda (event enrichment → target)
Custom App Event → Custom Event Bus → Multiple targets (fan-out)
```

### EventBridge Pipes

- **Point-to-point** integration with optional filtering and enrichment
- Source → (Filter) → (Enrichment via Lambda/Step Functions) → Target
- Sources: SQS, Kinesis, DynamoDB Streams, Kafka, MQ
- Less operational overhead than Lambda polling + forwarding

---

## Glue Workflows

- Orchestrate **Glue Crawlers and ETL Jobs** only (not other AWS services)
- Visual workflow builder in the console
- Trigger types: On-demand, scheduled, event-based (e.g., when another job completes)
- **Conditional triggers:** Start next job/crawler only if previous succeeded/failed
- Use Step Functions if you need to mix Glue with Lambda, EMR, Redshift, etc.

---

## Choosing the Right Orchestrator

| Scenario | Best Choice |
|----------|-------------|
| Multi-service AWS workflow with error handling | Step Functions |
| Need to wait for a long-running job (Glue, EMR) | Step Functions (`.sync:2`) |
| Human approval step in a pipeline | Step Functions (`.waitForTaskToken`) |
| Complex batch ETL with historical backfilling | MWAA |
| "Existing Airflow" migration to AWS | MWAA |
| Glue-only crawlers and jobs | Glue Workflows |
| React to S3 events, trigger pipeline | EventBridge |
| Scheduled recurring job (simple) | EventBridge Scheduler |
| "Serverless, no idle cost" orchestration | Step Functions |
| High-volume short executions (millions/day) | Step Functions Express |

---

## Exam Gotchas

- **Step Functions Standard vs Express:** Standard = long-running + audit trail; Express = high-throughput + short duration
- **`.sync:2`** — always use this when Step Functions needs to **wait** for a Glue/EMR job to complete
- **`.waitForTaskToken`** — use for human approval or external system callbacks
- **MWAA is NOT serverless** — costs money even when no DAGs run
- **Backfilling = MWAA/Airflow** — unique capability, no equivalent in Step Functions
- **Glue Workflows = Glue-only** — cannot call Lambda, EMR, Redshift directly
- **EventBridge replaces CloudWatch Events** — same service, same underlying infrastructure
- **EventBridge max event size = 256 KB** — larger payloads need S3 + pointer pattern
- **Archive/Replay** — EventBridge can replay past events after a bug fix
- **Cross-account events** — use custom event bus with resource policy
- `ResultPath: null` in Step Functions = discard task result, pass original input unchanged
