# Amazon QuickSight

Serverless business intelligence (BI) visualization service.

---

## Key Features

- **SPICE Engine:** Super-fast, Parallel, In-memory Calculation Engine
  - Imports data into in-memory storage for fast queries
  - Reduces load on data sources
- **Data Sources:** S3, Athena, Redshift, RDS, DynamoDB, and more
- **Sharing:** Dashboards, analyses, and datasets
- **ML Insights:** Anomaly detection, forecasting, auto-narratives

---

## User Types

| Type | Can Do |
|------|--------|
| **Author** | Create analyses, dashboards, datasets |
| **Reader** | View dashboards and receive emails |
| **Admin** | Manage users, data sources, account settings |

---

## Common Data Engineering Pattern

```
S3 (Parquet) -> Athena -> QuickSight (SPICE)
```

or

```
Redshift -> QuickSight (Direct Query or SPICE)
```

---

## Exam Gotchas

- QuickSight is the answer for "visualization" or "BI dashboards" on AWS
- SPICE = in-memory caching for fast dashboard performance
- QuickSight can connect directly to Athena for serverless end-to-end analytics
- Row-level security available for multi-tenant dashboards
