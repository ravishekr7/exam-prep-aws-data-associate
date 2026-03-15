# Amazon AppFlow

Fully managed integration service for transferring data between SaaS applications and AWS services.

---

## Key Features

- **No-code** data integration from SaaS sources
- Supports 50+ SaaS connectors (Salesforce, SAP, Google Analytics, Slack, ServiceNow, etc.)
- Destinations: S3, Redshift, Salesforce (bidirectional)
- Built-in data transformations: filtering, mapping, merging, masking
- Encryption in transit and at rest

---

## Flow Types

| Type | Description |
|------|-------------|
| **On-demand** | Triggered manually |
| **Scheduled** | Run at intervals (hourly, daily, etc.) |
| **Event-driven** | Triggered by source events (e.g., new Salesforce record) |

---

## When to Use

- "Transfer data from Salesforce to S3" = AppFlow
- "SaaS integration without custom code" = AppFlow
- "Ingest data from SAP to Redshift" = AppFlow

---

## Exam Gotchas

- AppFlow is the answer whenever SaaS data sources are mentioned
- Supports field-level encryption with customer-managed KMS keys
- Can trigger flows based on events from the source application
- Private data transfers via AWS PrivateLink (data doesn't traverse public internet)
