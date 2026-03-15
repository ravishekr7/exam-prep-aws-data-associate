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

## DynamoDB Accelerator (DAX)

- In-memory cache for DynamoDB
- **Microsecond** latency (vs millisecond for DynamoDB)
- Transparent to application (same API, no code changes)
- Write-through cache
- Use for read-heavy, latency-sensitive workloads

---

## DynamoDB Streams

- Captures item-level changes (INSERT, MODIFY, REMOVE)
- Ordered stream of changes per item
- Retention: 24 hours
- Common patterns:
  - Stream -> Lambda -> process changes
  - Stream -> Kinesis Data Streams (for longer retention or fan-out)

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

## Exam Gotchas

- **"Sub-second latency at scale"** = DynamoDB
- **"Microsecond latency"** = DAX
- GSI throttling can throttle the base table - always size GSI capacity appropriately
- LSI must be created at table creation time (cannot add later)
- On-Demand mode is more expensive per request but eliminates capacity planning
- **Scheduled Scaling** for predictable daily spikes (scale up at 8:55 AM, down at 10 AM)
- DynamoDB is NOT for analytics/aggregations (use Redshift/Athena for that)
- **Point-in-Time Recovery (PITR):** Restore to any second in the last 35 days
