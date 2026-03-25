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

## MWAA Environment Sizing

MWAA environments come in predefined sizes. Choosing the right size directly affects DAG parsing speed, scheduler responsiveness, and worker concurrency.

### Environment Classes
| Class | Max DAGs | Workers | Use Case |
|-------|----------|---------|----------|
| `mw1.small` | ~50 DAGs | 1-5 | Development, testing, few simple DAGs |
| `mw1.medium` | ~100 DAGs | 1-10 | Small production workloads |
| `mw1.large` | ~1000 DAGs | 1-25 | Standard production environments |
| `mw1.xlarge` | ~1000 DAGs | 1-50 | High-throughput environments |
| `mw1.2xlarge` | ~1000 DAGs | 1-100 | Very large enterprise workloads |

### Scheduler Configuration
- **Scheduler count:** Set to **2 for production** (High Availability). With a single scheduler, if it crashes, DAGs stop triggering until recovery.
- **Worker min/max:** MWAA auto-scales workers between your configured minimum and maximum based on the task queue depth.

### Performance Tip
If DAG parsing feels slow (DAGs take 10+ seconds to appear in the UI after S3 upload, or scheduler is slow to trigger runs):
- **Upgrade the environment class** — the scheduler CPU determines how fast it can parse and schedule DAGs
- Reduce the number of top-level DAG files (avoid importing heavy libraries at the module level)
- Use DAG serialization (enabled by default in Airflow 2.x) to reduce scheduler load

---

## DAG S3 Structure

MWAA reads DAGs and supporting files directly from S3. The bucket structure matters.

### Required Structure
```
s3://your-mwaa-bucket/
├── dags/                    # REQUIRED: all DAG .py files go here
│   ├── my_etl_dag.py
│   ├── data_quality_dag.py
│   └── replication_dag.py
├── plugins.zip              # OPTIONAL: custom Airflow plugins
└── requirements.txt         # OPTIONAL: additional Python packages
```

### requirements.txt
Install additional Python packages that your DAGs need:
```
# requirements.txt example
apache-airflow-providers-amazon>=8.0.0
pandas==2.0.3
requests==2.31.0
```

> **Warning:** Adding heavy packages (e.g., `tensorflow`, `torch`) can cause worker startup to slow significantly. Test in a dev environment first.

### Sync Delay
After you upload or modify a DAG file in S3, MWAA detects the change and loads it within approximately **30 seconds**. The DAG will appear (or update) in the Airflow UI shortly after. This is not instant — account for this delay in deployment pipelines.

---

## XCom vs S3 for Inter-Task Data

Airflow tasks can pass small values between each other using **XCom (Cross-Communication)**, but this has strict size limits.

### XCom (Cross-Communication)
- XCom values are stored in the **Airflow metadata database** (RDS/PostgreSQL behind MWAA)
- **Size limit: ~48 KB** (enforced by the database column type)
- Use for: task status strings, small JSON payloads, S3 file paths, database IDs

```python
# Task A: push a value to XCom
def push_file_path(**context):
    s3_path = "s3://bucket/output/2024-01-15/result.parquet"
    context['ti'].xcom_push(key='output_path', value=s3_path)

# Task B: pull the value from XCom
def consume_file_path(**context):
    s3_path = context['ti'].xcom_pull(task_ids='task_a', key='output_path')
    # Now read from s3_path
```

### Large Data: Always Use S3
If tasks need to share a large DataFrame, file, or result set:

1. **Task A** writes the data to S3 and pushes only the **S3 path** via XCom
2. **Task B** pulls the S3 path via XCom, then reads the data from S3

```python
# Task A: write large result to S3, share only the path
def process_and_store(**context):
    df = expensive_computation()  # 500 MB DataFrame
    s3_path = f"s3://bucket/tmp/{context['run_id']}/output.parquet"
    df.to_parquet(s3_path)
    context['ti'].xcom_push(key='result_path', value=s3_path)

# Task B: read from S3 path passed via XCom
def use_result(**context):
    s3_path = context['ti'].xcom_pull(task_ids='task_a', key='result_path')
    df = pd.read_parquet(s3_path)
```

**Exam pattern:** "Two Airflow tasks need to share a 500 MB DataFrame" → Write to S3 in Task A, pass the S3 path via XCom to Task B. Never try to push large data through XCom.

---

## MWAA vs Step Functions — Detailed Decision Guide

| Criteria | MWAA | Step Functions |
|----------|------|----------------|
| **Existing Airflow DAGs** | Lift-and-shift migration — use MWAA | Would require full rewrite |
| **Backfill historical data** | Built-in Airflow backfill command | No native backfill concept |
| **Task dependency complexity** | Hundreds of tasks, complex fan-out/in | Best for moderate workflows (<100 states) |
| **Python operators** | Any Python code, any library | Lambda only (no direct Python exec) |
| **Workflow duration** | Days to months (environment always running) | Standard: up to 1 year. Express: up to 5 min |
| **Cost model** | Pay per environment-hour (always on, even idle) | Pay per state transition (true serverless) |
| **Audit/execution history** | Airflow web UI with Gantt charts, logs per task | CloudWatch + Step Functions console (30-day history for Standard) |
| **AWS-native integration** | Via Boto3 or AWS operators (manual) | First-class SDK integrations (S3, Lambda, Glue, etc.) |

**Key exam tell:**
- "Team has **existing Airflow DAGs**" → MWAA
- "**Backfill** historical data" → MWAA (built-in Airflow concept)
- "**New** AWS-native workflow, no Airflow experience" → Step Functions
- "**Serverless** orchestration, no idle cost" → Step Functions

---

## Exam Gotchas

- MWAA is **NOT serverless** - you pay for the environment even when idle
- DAG files must be stored in S3
- "Existing Airflow migration" or "backfilling" = MWAA
- "Serverless orchestration" = Step Functions
- MWAA supports Python operators and can call any AWS service via Boto3
