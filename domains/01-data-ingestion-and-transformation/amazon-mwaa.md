# Amazon MWAA (Managed Workflows for Apache Airflow)

Managed version of open-source Apache Airflow for workflow orchestration.

---

## Key Concepts

- **DAGs (Directed Acyclic Graphs):** Python-defined workflows
- **Operators:** Tasks within a DAG (BashOperator, PythonOperator, AWS-specific operators)
- **Scheduler:** Triggers DAG runs based on schedule or events
- **Executor:** Runs tasks (CeleryExecutor in MWAA)

---

## When to Use MWAA vs Step Functions

| Feature | MWAA (Airflow) | Step Functions |
|---------|----------------|----------------|
| **Definition** | Python code (DAGs) | JSON/YAML (ASL) |
| **UI** | Rich web UI with Gantt charts, logs | Visual workflow editor |
| **Cost Model** | Pay for environment time (not serverless) | Pay per state transition |
| **Best For** | Complex dependencies, backfilling, existing Airflow teams | Event-driven, serverless workflows |
| **Setup Time** | Takes time to spin up environment | Instant |
| **Community** | Huge open-source ecosystem | AWS-native |

### Use MWAA When
- Migrating from existing Airflow
- Need complex scheduling with backfilling
- Need Python-based DAG logic with rich dependency management
- Team has Airflow expertise

### Use Step Functions When
- Building new AWS-native workflows
- Want serverless (no idle cost)
- Simple to moderate orchestration
- Need tight AWS service integration

---

## MWAA Architecture

- DAG files stored in **S3**
- Environment runs in your **VPC** (private or public web server)
- Logs sent to **CloudWatch**
- Supports custom plugins and requirements.txt for dependencies

---

## Exam Gotchas

- MWAA is **NOT serverless** - you pay for the environment even when idle
- DAG files must be stored in S3
- "Existing Airflow migration" or "backfilling" = MWAA
- "Serverless orchestration" = Step Functions
- MWAA supports Python operators and can call any AWS service via Boto3
