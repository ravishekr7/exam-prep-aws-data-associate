# Amazon QuickSight

Serverless business intelligence (BI) service for creating interactive dashboards, visualizations, and ML-powered insights at scale.

---

## SPICE Engine

**Super-fast, Parallel, In-memory Calculation Engine**

- Imports data into QuickSight's in-memory columnar storage
- Queries against SPICE do **not** hit the underlying data source → reduced cost and load
- SPICE data is replicated across AZs for high availability
- **Capacity:** Each account gets 10 GB free SPICE per author; additional capacity purchased per GB
- **Refresh:** Datasets can be refreshed on a schedule (hourly, daily, weekly) or via API
- **Incremental Refresh:** For supported sources (S3, Athena, Redshift), refresh only new/changed rows using a date/datetime column

### SPICE vs Direct Query

| Attribute | SPICE | Direct Query |
|---|---|---|
| Speed | Fast (in-memory) | Depends on source |
| Cost | SPICE capacity charges | Query charges on source (e.g., Athena per-scan) |
| Freshness | Stale until refresh | Always current |
| Data limit | SPICE capacity | No limit (query passes through) |
| Best for | Dashboards with many users | Real-time data, large datasets |

> **Exam tip:** Use SPICE when you need fast dashboards for many readers. Use Direct Query when data freshness is critical or dataset exceeds SPICE limits.

---

## Data Sources

Supported connections:
- **AWS:** S3 (CSV, JSON, Parquet, ORC, XLSX), Athena, Redshift, RDS/Aurora, DynamoDB, OpenSearch, Timestream
- **SaaS:** Salesforce, ServiceNow, Adobe Analytics, GitHub, Jira, Twitter
- **Databases:** MySQL, PostgreSQL, SQL Server, Oracle, Teradata, Presto
- **Upload:** Direct CSV/XLSX upload to SPICE

### VPC Connectivity

- QuickSight can connect to data sources inside a VPC using a **QuickSight VPC connection**
- Requires a VPC with a private subnet and appropriate security groups
- No need to expose databases publicly — keeps traffic within AWS network
- Supports Redshift, RDS, MySQL, PostgreSQL inside VPC

---

## Datasets

A **dataset** is a prepared, reusable data object in QuickSight. It defines:
- Which data source to connect to
- Which tables/fields to include
- Joins between tables
- Calculated fields and transformations
- Row-level security rules

### Dataset Joins

- Supports INNER, LEFT, RIGHT, FULL OUTER joins between tables (from the same or different data sources)
- Join columns must be of compatible data types
- For SPICE datasets, joins are evaluated at import time
- For Direct Query, joins run at query time against the source

### Calculated Fields

Custom metrics derived from existing fields using QuickSight functions:

```
# Example: Profit margin
{Revenue} - {Cost}

# Date formatting
formatDate({order_date}, "yyyy-MM-dd")

# Conditional logic
ifelse({status} = "completed", "Done", "Pending")

# Aggregated calculated field (used in visuals)
sum({revenue}) / countDistinct({customer_id})
```

- Can reference other calculated fields
- **Table calculations** apply window functions (running totals, rank, percentile) — evaluated after aggregation

### Data Transformations

- Rename fields, change data types, hide fields
- Filter rows at dataset level (reduces data imported to SPICE)
- Exclude nulls, replace values
- Create date hierarchies (year → quarter → month → day)

---

## Visual Types

| Visual | Best For |
|---|---|
| **Bar Chart** (horizontal/vertical) | Category comparisons |
| **Line Chart** | Trends over time |
| **Pie / Donut Chart** | Part-to-whole (small # of categories) |
| **Scatter Plot** | Correlation between two measures |
| **Heatmap** | Two-dimensional frequency/intensity |
| **Tree Map** | Hierarchical part-to-whole |
| **Pivot Table** | Multi-dimensional aggregations |
| **KPI** | Single metric vs target with trend |
| **Gauge** | Progress toward a goal |
| **Geospatial Map** | Location-based data |
| **Funnel Chart** | Stage-based conversion |
| **Box Plot** | Statistical distribution |
| **Word Cloud** | Text frequency |
| **Combo Chart** | Bar + line on same axis |
| **Waterfall Chart** | Sequential positive/negative changes |

### Aggregation Functions

`sum`, `avg`, `min`, `max`, `count`, `countDistinct`, `median`, `percentile`, `stdev`, `variance`

---

## Dashboard Features

### Controls (Filters)

- **Filter controls:** Dropdown, multi-select, date range, numeric sliders applied to visuals
- **Parameters:** Named variables that can drive filters, calculated fields, or URLs
  - Example: A `selectedYear` parameter drives a date filter across all visuals
- **Cascading filters:** Selecting one control narrows options in another (e.g., Region → State)

### Sheet Organization

- Dashboards can have **multiple sheets** (tabs)
- Each sheet has independent layout and visuals
- Sheets can share the same dataset or use different datasets

### Dashboard Sharing

- Dashboards are **published snapshots** of an analysis — readers cannot edit
- Share with:
  - Individual QuickSight users/groups
  - **Public embedding** (via signed URL — for external users)
  - **Email reports** (scheduled PDF/CSV delivery to readers)
- Readers do not need an AWS account — they just need a QuickSight Reader license

### Embedding

- Embed dashboards in web apps using the **QuickSight Embedding SDK**
- Two modes:
  - **1-Click Embed (Anonymous):** No login — generates a signed URL for anonymous access
  - **Registered User Embed:** User logs in via QuickSight identity or SSO
- Use `GenerateEmbedUrlForAnonymousUser` or `GenerateEmbedUrlForRegisteredUser` API calls
- **Runtime Filters:** Pass filter values at embed time (e.g., show only data for `tenant_id=123`)

---

## Row-Level Security (RLS)

Controls which rows of data each user/group can see within the same dataset.

### How It Works

1. Create a **permissions dataset** — a table mapping users/groups to allowed dimension values
2. Apply RLS to a QuickSight dataset, referencing the permissions dataset
3. QuickSight automatically filters data at query time based on the user viewing the dashboard

### Permissions Dataset Format

```
UserName (or GroupName) | allowed_region | allowed_product
alice@example.com       | US-East        | ProductA
bob@example.com         | EU-West        | ProductB
```

- `UserName` column matches the QuickSight username
- Additional columns match field names in the main dataset
- A user with no row in the permissions dataset sees **no data**
- Wildcards (`*`) in a column value mean the user can see all values for that field

### Tag-Based RLS

- For **anonymous embedding**, use **Tag-Based RLS** — pass tag values at embed time
- Tags are key-value pairs injected into the embed URL generation call
- More scalable than managing explicit user rows in a permissions dataset

> **Exam tip:** RLS is critical for multi-tenant dashboards. Tag-based RLS is the answer for anonymous embed scenarios where you can't pre-register users.

---

## Column-Level Security (CLS)

- Restrict specific columns from being seen by certain users/groups
- Configured at the **dataset level** via column-level permissions
- Useful for hiding PII fields (SSN, salary) from certain roles
- Columns hidden by CLS cannot be used in calculated fields by restricted users

---

## ML Insights

### Anomaly Detection

- Automatically identifies unusual data points in time-series or categorical data
- Uses Random Cut Forest (RCF) algorithm under the hood
- Configure: contribution analysis to identify which dimensions drive the anomaly
- **Anomaly Alert:** QuickSight can email users when anomalies exceed a threshold

### Forecasting

- Add a forecast to any line chart (time-series visual)
- Configurable forecast period and prediction interval (e.g., 80% confidence band)
- Handles seasonality automatically
- Point it at a date dimension + measure — no ML expertise required

### Auto-Narratives

- QuickSight generates plain-English summaries of charts ("Sales increased 12% in Q3 vs Q2, driven by Product A")
- Appears as a text box next to the visual
- Customizable phrasing templates

### QuickSight Q (Natural Language Queries)

- Users type questions in plain English ("What were sales by region last quarter?")
- Q maps natural language to dataset fields using a **Topic** configuration
- **Topics:** An author defines friendly names, synonyms, and relationships for fields
  - Example: Map "revenue" synonym to the `total_order_value` field
- Results appear as auto-generated visuals
- Q requires a Business or Enterprise subscription

### QuickSight Generative BI (new)

- Uses generative AI to answer questions, build dashboards from prompts, and explain visuals
- "Build me a dashboard for sales performance by region" → auto-generates visuals

---

## User Management & Licensing

| License | Can Do | Cost Model |
|---|---|---|
| **Admin** | Full account control, user management, data source setup | Author pricing |
| **Author** | Create/edit analyses, datasets, dashboards | Per-author/month |
| **Reader** | View published dashboards, use Q | Per-session or per-reader/month |

### Reader Pricing Models

- **Per-session pricing:** Pay per 30-minute session (good for infrequent users)
- **Per-reader pricing:** Fixed monthly cost (good for users who access dashboards daily)

> **Exam tip:** Use per-session pricing for large user pools with infrequent access. Use per-reader for power users who check dashboards daily. Authors are more expensive — only create author accounts for people building dashboards.

### Identity Providers

- **QuickSight-managed users:** Account managed within QuickSight
- **IAM Federation:** Map IAM roles to QuickSight users
- **SSO (IAM Identity Center):** Enterprise SSO integration
- **Active Directory:** Integrate with AWS Directory Service or on-premises AD

---

## Performance Optimization

### SPICE Performance Tips

- Import data to SPICE vs Direct Query for high-concurrency dashboards
- Use **incremental refresh** to reduce refresh time on large datasets
- Pre-aggregate data in the source (Redshift/Athena materialized views) before importing to SPICE
- Filter rows at dataset level to minimize SPICE storage usage

### Direct Query Performance Tips

- Use Athena partitioned tables to minimize data scanned per query
- Use Redshift materialized views for pre-computed aggregations
- Enable **query caching** on Athena workgroup to reuse identical query results

### Dashboard Load Time

- Limit the number of visuals per sheet (< 20 per sheet recommended)
- Use **sheet-level filters** instead of visual-level filters to apply once
- Pre-filter data in the dataset rather than in each visual

---

## Common Patterns

### Pattern 1: Serverless Analytics Dashboard

```
S3 (Parquet, partitioned) → Athena (Workgroup) → QuickSight SPICE
                                                      ↓
                                              Scheduled daily refresh
```

### Pattern 2: Real-Time BI on Redshift

```
Kinesis Firehose → Redshift → QuickSight (Direct Query)
```
Use Direct Query so dashboards always reflect the latest data without a refresh cycle.

### Pattern 3: Multi-Tenant SaaS Dashboard (RLS)

```
App → QuickSight Embed (signed URL with tenant_id tag)
            ↓
   Tag-Based RLS filters dataset to tenant's rows only
```

### Pattern 4: Self-Service Analytics with Q

```
Redshift → QuickSight Dataset + Topic Definition → Business Users ask questions in Q
```

### Pattern 5: Operational Cost Dashboard

```
AWS Cost & Usage Report → S3 → Athena → QuickSight SPICE
```

---

## Integration with Athena

- Most common QuickSight data source for data lakes
- QuickSight uses **Athena workgroups** — configure the workgroup in the data source connection
- Bytes scanned by Athena for QuickSight queries count toward Athena costs
- SPICE refresh triggers Athena queries → minimize refresh frequency for large tables or use incremental refresh
- Use Athena partition projection to avoid Glue catalog overhead on every refresh

---

## Security

| Feature | Details |
|---|---|
| **Encryption at rest** | SPICE data encrypted at rest with AWS-managed keys |
| **Encryption in transit** | TLS for all connections |
| **KMS integration** | Bring your own KMS key for SPICE encryption |
| **VPC** | Connect to sources in private VPC via VPC connection |
| **RLS / CLS** | Row and column level data access control |
| **IAM** | Control who can manage QuickSight resources |
| **CloudTrail** | QuickSight API calls logged to CloudTrail |

---

## Exam Gotchas

- **QuickSight = visualization/BI dashboard answer** on the exam — not Athena, not Redshift
- **SPICE** = in-memory caching for fast dashboards at scale
- **RLS** is the answer for "only show each user their own data" scenarios
- **Tag-based RLS** is used for anonymous embedding (not regular RLS which requires user rows)
- **Q Topics** must be configured before natural language queries work
- **Readers don't pay per query** — per-session or per-reader pricing regardless of queries run
- **SPICE capacity is per account** — plan capacity when onboarding many authors
- **Incremental refresh** requires a date column to identify new/changed records
- **Embedding requires signed URLs** — never expose QuickSight credentials to the browser
- **Column-level security** cannot be circumvented via calculated fields — restricted columns are completely hidden
- QuickSight **cannot directly query DynamoDB** without going through an intermediary (Athena → DynamoDB connector, or export to S3)
