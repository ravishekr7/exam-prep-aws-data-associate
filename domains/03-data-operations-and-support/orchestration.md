# Orchestration

Coordinating workflows across multiple AWS services.

---

## Service Comparison

| Feature | Step Functions | MWAA (Airflow) | Glue Workflows | EventBridge |
|---------|---------------|----------------|----------------|-------------|
| **Type** | State machine | DAG orchestrator | ETL workflow | Event router |
| **Serverless** | Yes | No (pay for env) | Yes | Yes |
| **Definition** | JSON/YAML (ASL) | Python (DAGs) | Visual/API | Event rules |
| **Best For** | AWS service orchestration | Complex dependencies, backfilling | Glue-only workflows | Event-driven triggers |

---

## Amazon EventBridge

- Serverless event bus for connecting applications
- **Rules:** Match events and route to targets
- **Targets:** Lambda, Step Functions, SQS, SNS, Kinesis, ECS, and more
- **Schedulers:** Cron or rate-based schedules (replaced CloudWatch Events)
- **Schema Registry:** Discover and manage event schemas

### Common Patterns
- S3 event -> EventBridge -> Step Functions (start workflow)
- Schedule -> EventBridge -> Glue Job (scheduled ETL)
- Custom app event -> EventBridge -> Lambda (event processing)

---

## Glue Workflows

- Orchestrate Glue crawlers and ETL jobs
- Visual workflow builder
- Trigger-based execution
- Limited to Glue services only (use Step Functions for multi-service)

---

## Choosing the Right Orchestrator

| Scenario | Best Choice |
|----------|-------------|
| Multi-service AWS workflow with error handling | Step Functions |
| Complex batch ETL with backfilling needs | MWAA |
| Glue-only crawlers and jobs | Glue Workflows |
| Event-driven routing and scheduling | EventBridge |
| "Existing Airflow" migration | MWAA |
| "Serverless orchestration" | Step Functions |

---

## Exam Gotchas

- Step Functions + EventBridge is a powerful serverless combo
- MWAA is NOT serverless - costs money even when idle
- Glue Workflows are limited to Glue services only
- EventBridge replaces CloudWatch Events (same underlying service)
- "Backfilling" = MWAA/Airflow (unique capability)
