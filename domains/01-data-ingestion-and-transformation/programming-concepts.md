# Programming Concepts and Infrastructure as Code

Covers programming, IaC, CI/CD, and software engineering concepts for the exam.

---

## Programming Languages in Scope

| Language | Where Used |
|----------|-----------|
| **Python** | Glue ETL, Lambda, MWAA DAGs, EMR (PySpark) |
| **SQL** | Athena, Redshift, Glue (SparkSQL), RDS |
| **Scala** | Glue ETL, EMR (Spark) |
| **Java** | EMR, Kinesis KPL/KCL |
| **Bash** | EMR bootstrap actions, automation scripts |
| **R** | SageMaker, EMR |

---

## Infrastructure as Code (IaC)

### AWS CloudFormation
- YAML/JSON templates for AWS resource provisioning
- Stack-based deployment and rollback
- Drift detection for configuration changes

### AWS CDK (Cloud Development Kit)
- Write infrastructure in Python, TypeScript, Java, C#
- Synthesizes to CloudFormation templates
- Higher-level constructs for common patterns

### AWS SAM (Serverless Application Model)
- Extension of CloudFormation for serverless
- Package and deploy: Lambda functions, Step Functions, DynamoDB tables, API Gateway
- Local testing with `sam local`

---

## CI/CD Pipeline

```
CodeCommit/GitHub -> CodeBuild (build/test) -> CodeDeploy (deploy) -> CodePipeline (orchestrates all)
```

| Service | Role |
|---------|------|
| **CodePipeline** | Orchestrates the full CI/CD pipeline |
| **CodeBuild** | Build and test code (compile, run unit tests) |
| **CodeDeploy** | Deploy to EC2, Lambda, ECS |

---

## Distributed Computing Concepts

- **MapReduce:** Split data across nodes (Map), aggregate results (Reduce)
- **Partitioning:** Distribute data by key for parallel processing
- **Shuffling:** Redistribute data across nodes (expensive in Spark)
- **DAG (Directed Acyclic Graph):** Execution plan for distributed jobs (Spark, Airflow)

---

## Data Structures (In Scope)

- **Graph structures:** Used in Neptune, knowledge graphs, lineage tracking
- **Tree structures:** Used in hierarchical data, JSON/XML parsing
- **Hash tables:** DynamoDB partition key hashing for data distribution

---

## Exam Gotchas

- Know that CDK synthesizes to CloudFormation (not a replacement, an abstraction)
- SAM is specifically for serverless deployments
- "Repeatable infrastructure deployment" = CloudFormation or CDK
- "Package and deploy Lambda" = SAM
- Version control, testing, and logging are expected best practices
- Understand distributed computing concepts (don't need to write code)
