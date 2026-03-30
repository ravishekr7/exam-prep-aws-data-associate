# Amazon DynamoDB

Key-value and document NoSQL database. Single-digit millisecond latency at any scale. OLTP.

---

## Capacity Modes

| Mode | Description | Best For |
|------|-------------|----------|
| **On-Demand** | Pay per request | Unpredictable/spikey traffic, development |
| **Provisioned** | Specify WCU/RCU, supports Auto-Scaling | Predictable load, cost optimization |

- **WCU (Write Capacity Unit):** 1 write of up to 1 KB per second
- **RCU (Read Capacity Unit):** 1 strongly consistent read of up to 4 KB per second (or 2 eventually consistent reads)

---

## RCU/WCU Calculation

> **This is heavily tested. Master the formulas and be able to calculate in your head.**

### Formulas

**WCU (Write Capacity Unit):**
```
WCUs per write = ceil(item_size_KB / 1)
Transactional write = 2 × standard WCU
```

**RCU (Read Capacity Unit):**
```
Strong Consistency:    RCUs per read = ceil(item_size_KB / 4)
Eventual Consistency:  RCUs per read = ceil(item_size_KB / 4) / 2   (half the cost)
Transactional read:    RCUs per read = 2 × strong consistency RCU
```

### Worked Examples

| Item Size | Write (WCU) | Transactional Write | Strong Read (RCU) | Eventual Read (RCU) | Transactional Read |
|-----------|-------------|--------------------|--------------------|---------------------|--------------------|
| **1 KB** | ceil(1/1) = **1 WCU** | 2 WCU | ceil(1/4) = **1 RCU** | 1/2 = **0.5 RCU** | 2 RCU |
| **6 KB** | ceil(6/1) = **6 WCU** | 12 WCU | ceil(6/4) = **2 RCU** | 2/2 = **1 RCU** | 4 RCU |
| **10 KB** | ceil(10/1) = **10 WCU** | 20 WCU | ceil(10/4) = **3 RCU** | 3/2 = **1.5 RCU** | 6 RCU |

### Throughput Calculation Example
**Scenario:** 100 reads/second of 6 KB items with eventual consistency. How many RCUs needed?
```
RCU per read (eventual) = ceil(6/4) / 2 = 2/2 = 1 RCU
Total RCUs = 100 reads/sec × 1 RCU = 100 RCUs provisioned
```

**Exam trap:** DynamoDB rounds up item size to the next 4 KB boundary for reads and next 1 KB for writes.

---

## Key Design

### Primary Key Options
1. **Partition Key only:** Must be unique per item
2. **Composite Key (Partition Key + Sort Key):** Combination must be unique

### Key Selection Best Practices
- Partition key should have **high cardinality** for even distribution
- Sort key enables range queries (`>`, `<`, `begins_with`, `between`)

---

## Secondary Indexes

| Feature | LSI (Local Secondary Index) | GSI (Global Secondary Index) |
|---------|---------------------------|------------------------------|
| **Alternative** | Different Sort Key | Different Partition Key + Sort Key |
| **Creation** | Table creation time ONLY | Anytime |
| **Capacity** | Shares base table WCU/RCU | Has its own WCU/RCU |
| **Consistency** | Supports strong consistency | Eventually consistent only |
| **Limit** | 5 per table | 20 per table |

### GSI Throttling
If GSI runs out of WCU, **base table writes are throttled** too. Always provision enough capacity for GSIs.

---

## GSI vs LSI Deep Comparison

| Aspect | LSI (Local Secondary Index) | GSI (Global Secondary Index) |
|--------|---------------------------|------------------------------|
| **When created** | Must be created at **table creation time** — cannot add later | Can be added to existing tables **at any time** |
| **Partition key** | Same partition key as the base table | Can be **any attribute** — different from base table PK |
| **Sort key** | Must be different from the base table sort key | Can be any attribute (or none) |
| **Consistency model** | Supports both strong and eventual consistency | **Eventual consistency only** — cannot do strongly consistent reads on a GSI |
| **Throughput** | **Shares** the base table's provisioned WCU/RCU | Has **its own independent** WCU/RCU (must provision separately) |
| **Storage** | Counts toward base table storage | Has its own storage allocation |
| **Count limit** | Max **5 LSIs** per table | Max **20 GSIs** per table |
| **Primary use case** | Alternative sort order on the same partition (e.g., user's orders by date instead of order ID) | Query patterns that require a different partition key entirely (e.g., find all orders for a product, not just a user) |

### Key Exam Facts
- **LSI cannot be added after table creation.** If you realize you need an LSI after the table exists, you must recreate the table. This is a critical design decision.
- **GSI supports only eventual consistency.** If a question requires strongly consistent reads on a secondary access pattern → use LSI (if partition key matches) or rethink the data model.
- **GSI write throttling cascades** to the base table — if the GSI's WCUs are exhausted, writes to the base table fail even if base table WCUs are available.

---

## DynamoDB Accelerator (DAX)

- In-memory cache for DynamoDB
- **Microsecond** latency (vs millisecond for DynamoDB)
- Transparent to application (same API, no code changes)
- **Read cache** — writes pass through DAX to DynamoDB directly, but DAX provides zero write latency benefit. Do not use DAX to solve write performance problems.
- Use for read-heavy, latency-sensitive workloads

---

## DynamoDB Streams

Captures item-level changes (INSERT, MODIFY, REMOVE) as an ordered stream of events per partition key.

- **Retention:** 24 hours — records are available for 24 hours after being written
- **Ordering:** Guaranteed per partition key, not globally across the table
- **Common integrations:**
  - Streams → Lambda (real-time processing)
  - Streams → Kinesis Data Streams (longer retention up to 365 days, fan-out to multiple consumers)

### Stream Record Content
Each stream record contains:
- **Event type:** `INSERT`, `MODIFY`, or `REMOVE`
- **OLD_IMAGE:** The item's state before the change (available for MODIFY and REMOVE)
- **NEW_IMAGE:** The item's state after the change (available for INSERT and MODIFY)
- Configure stream view type: `KEYS_ONLY`, `NEW_IMAGE`, `OLD_IMAGE`, `NEW_AND_OLD_IMAGES`

### Lambda Filter Policies
Instead of Lambda processing every record and discarding irrelevant ones, you can configure **event source filter policies** on the Lambda trigger. Lambda only receives records matching your filters — reducing invocations and cost.

```json
{
  "Filters": [
    {
      "Pattern": {
        "eventName": ["INSERT"],
        "dynamodb": {
          "NewImage": {
            "status": { "S": ["PENDING"] }
          }
        }
      }
    }
  ]
}
```

### Fan-Out Pattern
```
DynamoDB Streams → Lambda → SNS topic → multiple SQS queues
                                        → SQS Queue A (fulfillment service)
                                        → SQS Queue B (notification service)
                                        → SQS Queue C (analytics service)
```

This decouples downstream consumers — each processes at its own rate without blocking others.

### Exactly-Once Processing
DynamoDB Streams + Lambda uses **at-least-once delivery**. Lambda may invoke your function more than once for the same record (due to retries). Design your Lambda functions to be **idempotent** — processing the same record twice should produce the same result.

Common patterns for idempotency:
- Conditional writes (`ConditionExpression`) in DynamoDB
- Check if the operation was already applied (e.g., check a `processed_at` attribute)
- Unique constraint checks in the destination system

---

## Global Tables

- Multi-region, multi-active replication
- Sub-second replication between regions
- Active-active: read and write in any region
- Requires DynamoDB Streams enabled
- Use for: global applications, disaster recovery

---

## TTL (Time to Live)

- Automatically delete items after expiration timestamp
- No WCU cost for TTL deletions
- Deletions appear in DynamoDB Streams
- Use for: session data, temporary records, log expiration

---

## Item Size Limit

Each DynamoDB item has a hard maximum size of **400 KB** — including all attribute names and values.

**Exam trap:** If a scenario involves storing large objects (images, videos, PDFs, large JSON documents) in DynamoDB, the answer is always:
1. Store the object in **S3**
2. Store the **S3 object key** as a DynamoDB attribute

You cannot exceed 400 KB per item — there is no way to configure or raise this limit.

---

## Conditional Writes (ConditionExpression)

DynamoDB supports conditional writes — a write only succeeds if a condition on the current item state is true. If the condition fails, DynamoDB returns a `ConditionalCheckFailedException` and the item is not modified.

**Common patterns:**

```python
import boto3
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Orders')

# Pattern 1: Write only if item does NOT already exist (prevent overwrite)
table.put_item(
    Item={'order_id': '123', 'status': 'PENDING', 'version': 1},
    ConditionExpression='attribute_not_exists(order_id)'
)

# Pattern 2: Optimistic locking — update only if version matches expected
table.update_item(
    Key={'order_id': '123'},
    UpdateExpression='SET #s = :new_status, version = :new_version',
    ConditionExpression='version = :expected_version',
    ExpressionAttributeNames={'#s': 'status'},
    ExpressionAttributeValues={
        ':new_status': 'SHIPPED',
        ':new_version': 2,
        ':expected_version': 1   # fails if another process already incremented this
    }
)
```

**Exam Patterns:**
- "Prevent two concurrent processes from overwriting each other's DynamoDB writes" → `ConditionExpression` with a `version` attribute (optimistic locking)
- "Write a new item only if it doesn't already exist" → `attribute_not_exists(pk)` in `ConditionExpression`
- "Implement optimistic locking in DynamoDB" → `ConditionExpression` checking a version counter

---

## DynamoDB Transactions

`TransactWriteItems` and `TransactGetItems` provide all-or-nothing semantics across multiple items — even across multiple tables. If any operation in the transaction fails its condition, the entire transaction is rolled back.

**Cost:** Transactional operations consume **2× the normal WCU/RCU** (as shown in the capacity formulas above).

**TransactWriteItems** — up to 100 items, up to 4 MB total:
```python
client = boto3.client('dynamodb')

# Atomically debit one account and credit another
client.transact_write_items(
    TransactItems=[
        {
            'Update': {
                'TableName': 'Accounts',
                'Key': {'account_id': {'S': 'A001'}},
                'UpdateExpression': 'SET balance = balance - :amount',
                'ConditionExpression': 'balance >= :amount',  # fail if insufficient funds
                'ExpressionAttributeValues': {':amount': {'N': '100'}}
            }
        },
        {
            'Update': {
                'TableName': 'Accounts',
                'Key': {'account_id': {'S': 'A002'}},
                'UpdateExpression': 'SET balance = balance + :amount',
                'ExpressionAttributeValues': {':amount': {'N': '100'}}
            }
        }
    ]
)
# If A001 has insufficient balance, BOTH updates are rolled back
```

**Exam Patterns:**
- "Update two DynamoDB items atomically — either both succeed or neither does" → `TransactWriteItems`
- "Write to two different DynamoDB tables as a single atomic operation" → `TransactWriteItems` (can span tables)
- "Transactional writes cost 2× WCU" — exam will test this in capacity calculation questions

---

## DynamoDB Export to S3

DynamoDB's Export to S3 feature creates a point-in-time snapshot of your table and writes it to S3 — **without consuming any read capacity**.

### Key Facts
- **Does NOT consume RCUs** — unlike a Scan operation, which reads every item and charges RCUs
- Output formats: **DynamoDB JSON** (native format) or **Amazon Ion** (structured data format)
- **Point-in-time export:** Exports data as of a specific timestamp in the past (up to 35 days back). Requires **PITR (Point-in-Time Recovery)** to be enabled on the table.
- Data exported to S3 is partitioned by timestamp for easy ingestion

### Use Cases
- **Compliance snapshots:** Export table state at a specific point in time for audit/compliance
- **Feeding data lakes:** Regular exports to S3 → Glue Crawler → Athena queries for analytics without impacting production DynamoDB performance
- **Large-scale analytics:** Run Spark/Athena on the exported data instead of scanning the live table (which is expensive and slow)
- **Initial data load:** Export existing table to S3 before migrating to a new system

```python
# Trigger export via Boto3
import boto3
from datetime import datetime
client = boto3.client('dynamodb')

response = client.export_table_to_point_in_time(
    TableArn='arn:aws:dynamodb:us-east-1:123456789:table/Orders',
    S3Bucket='my-data-lake-bucket',
    S3Prefix='dynamodb-exports/orders/',
    ExportFormat='DYNAMODB_JSON',   # or 'ION'
    ExportTime=datetime(2024, 1, 15, 0, 0, 0)  # optional: specific point in time
)
```

---

## Parallel Scan

By default, a DynamoDB Scan reads the entire table sequentially from a single worker. For large tables this is slow, expensive (consumes RCUs), and can take hours.

Parallel Scan divides the table into N segments and scans each segment simultaneously using separate workers. Total scan time is reduced proportionally to the number of segments.

**How it works:**

Each worker is assigned a segment number (0 to TotalSegments-1). Each worker scans only its assigned segment, independently of the others. Results from all segments are then combined by the application.

```python
import boto3
import threading

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('LargeOrdersTable')

TOTAL_SEGMENTS = 10  # number of parallel workers
results = []
lock = threading.Lock()

def scan_segment(segment):
    response = table.scan(
        TotalSegments=TOTAL_SEGMENTS,
        Segment=segment,            # this worker only scans this segment
        ProjectionExpression='order_id, amount, status'
    )
    items = response['Items']
    while 'LastEvaluatedKey' in response:
        response = table.scan(
            TotalSegments=TOTAL_SEGMENTS,
            Segment=segment,
            ExclusiveStartKey=response['LastEvaluatedKey'],
            ProjectionExpression='order_id, amount, status'
        )
        items.extend(response['Items'])

    with lock:
        results.extend(items)

# Launch all segments in parallel
threads = [threading.Thread(target=scan_segment, args=(i,)) for i in range(TOTAL_SEGMENTS)]
for t in threads: t.start()
for t in threads: t.join()

print(f"Total items scanned: {len(results)}")
```

**Key considerations:**

- **RCU cost:** Parallel Scan consumes RCUs just like a sequential scan — scanning the whole table is expensive regardless of parallelism. Use DynamoDB Export to S3 instead if you need to process the whole table for analytics (Export does NOT consume RCUs).

- **When to use Parallel Scan vs Export to S3:**

| Scenario | Use |
|---|---|
| One-time full table analysis for analytics | DynamoDB Export to S3 (no RCU cost) |
| Real-time application needs to scan table quickly with custom logic | Parallel Scan |
| Regular analytics workload on large table | Export to S3 → Athena/Glue (cheaper, scalable) |
| Need exactly point-in-time snapshot | Export to S3 with PITR timestamp |

- **Provisioned capacity impact:** Each scan segment consumes RCUs. With 10 parallel segments, you consume capacity 10x faster. For provisioned tables, this can exhaust RCUs and throttle other operations. Use exponential backoff and limit parallelism for provisioned tables.

**Exam Patterns:**
- "Scan a large DynamoDB table as fast as possible in a one-off migration" → Parallel Scan
- "Reduce DynamoDB scan time from 2 hours to 15 minutes for a nightly process" → Parallel Scan with multiple segments
- "Full table scan for analytics without consuming read capacity" → DynamoDB Export to S3 (Parallel Scan consumes RCUs; Export does not)

---

## Exam Gotchas

- **"Single-digit millisecond latency at any scale"** = DynamoDB (not "sub-second" — that's too vague and matches many services)
- **"Microsecond latency"** = DAX
- GSI throttling can throttle the base table - always size GSI capacity appropriately
- LSI must be created at table creation time (cannot add later)
- On-Demand mode is more expensive per request but eliminates capacity planning
- **Scheduled Scaling** for predictable daily spikes (scale up at 8:55 AM, down at 10 AM)
- DynamoDB is NOT for analytics/aggregations (use Redshift/Athena for that)
- **Point-in-Time Recovery (PITR):** Restore to any second in the last 35 days
- **On-Demand mode** automatically scales to handle any traffic level without capacity planning. Cost per request is higher than provisioned mode. At sustained high throughput (millions of requests/hour), provisioned mode with Auto Scaling is significantly cheaper.
- **Provisioned mode with Auto Scaling:** Define min/max WCU/RCU and a target utilization (e.g., 70%). DynamoDB Auto Scaling adjusts capacity in response to actual traffic. Best for workloads with gradual, predictable ramps.
- **DAX (DynamoDB Accelerator):** Provides microsecond read latency via an in-memory cache. DAX is a read cache — it does **not** help write-heavy workloads. Also: DAX only serves **eventually consistent reads**. If your application requires strongly consistent reads, bypass DAX and read from DynamoDB directly.
- **DynamoDB TTL** deletes expired items asynchronously — items may persist for up to **48 hours** after their TTL expiry timestamp. Do not rely on TTL for time-critical deletions. TTL deletions do NOT consume WCUs.
- **DynamoDB item size limit is 400 KB** — hard limit, not configurable. Large objects (images, documents) must be stored in S3; store the S3 key in DynamoDB.
- **ConditionExpression prevents lost updates** — use `attribute_not_exists(pk)` to prevent overwriting existing items, and a `version` counter for optimistic locking. If the condition fails, DynamoDB throws `ConditionalCheckFailedException` and makes no change.
- **TransactWriteItems is all-or-nothing across up to 100 items and multiple tables.** Cost is 2× normal WCU. Use when two or more writes must either all succeed or all fail together.
- **Parallel Scan consumes RCUs — DynamoDB Export to S3 does not.** This is the key decision: if cost matters and you need the full table, use Export to S3. Use Parallel Scan only when you need real-time programmatic access to the data and cannot use the Export approach.
- **Parallel Scan segments must be processed by separate workers.** A single-threaded application scanning TotalSegments=10 with Segment=0 only scans 1/10 of the table. You need 10 separate threads/processes/Lambda invocations to get the parallelism benefit.
