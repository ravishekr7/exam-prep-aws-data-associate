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

## Integration Patterns

- **Request-Response:** Call service, get immediate response (default)
- **Run a Job (.sync):** Wait for the job to complete (Glue, EMR, ECS)
- **Wait for Callback (.waitForTaskToken):** Pause until external system calls back

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
