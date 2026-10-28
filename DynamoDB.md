# Amazon DynamoDB — Master Notes

> Consolidated, interview-ready notes on Amazon DynamoDB. Formatting standardized: `##` for top-level topics, `###`/`####` for sub-sections, tables for comparisons.

## Table of Contents

1. [What is DynamoDB?](#1-what-is-dynamodb)
2. [Core Concepts: Tables, Items, Attributes](#2-core-concepts-tables-items-attributes)
3. [Primary Keys](#3-primary-keys)
4. [Partitions & How Data Is Distributed](#4-partitions--how-data-is-distributed)
5. [Capacity Modes: Provisioned vs On-Demand](#5-capacity-modes-provisioned-vs-on-demand)
6. [Calculating RCUs and WCUs](#6-calculating-rcus-and-wcus)
7. [Read Consistency Models](#7-read-consistency-models)
8. [Query vs Scan](#8-query-vs-scan)
9. [Secondary Indexes: LSI vs GSI](#9-secondary-indexes-lsi-vs-gsi)
10. [Data Modeling & Single-Table Design](#10-data-modeling--single-table-design)
11. [Write Operations, Conditional Writes & Batching](#11-write-operations-conditional-writes--batching)
12. [Transactions](#12-transactions)
13. [Hot Partitions, Throttling & Retries](#13-hot-partitions-throttling--retries)
14. [DynamoDB Streams](#14-dynamodb-streams)
15. [Time to Live (TTL)](#15-time-to-live-ttl)
16. [Global Tables](#16-global-tables)
17. [DynamoDB Accelerator (DAX)](#17-dynamodb-accelerator-dax)
18. [Backup, Restore & Import/Export](#18-backup-restore--importexport)
19. [Security](#19-security)
20. [Integrations with Other AWS Services](#20-integrations-with-other-aws-services)
21. [Monitoring & Observability](#21-monitoring--observability)
22. [DynamoDB vs Other Databases](#22-dynamodb-vs-other-databases)
23. [Pricing & Cost Optimization](#23-pricing--cost-optimization)
24. [Limits Quick Reference](#24-limits-quick-reference)
25. [Best Practices](#25-best-practices)
26. [Common Interview Questions](#26-common-interview-questions)

---

## 1. What is DynamoDB?

**Amazon DynamoDB** is a fully managed, serverless **NoSQL key-value and document database** that delivers consistent **single-digit millisecond** performance at virtually any scale.

### Key Characteristics
- **Serverless**: no servers, patching, or instances to manage; capacity is either provisioned or fully on-demand.
- **Highly available and durable**: data is automatically replicated across **three Availability Zones** in a Region (99.99% SLA for single-Region tables, 99.999% for global tables).
- **Schema-flexible**: only the primary key attributes are required; every other attribute can vary item to item.
- **Access-pattern driven**: DynamoDB is designed for **known, predictable access patterns** (lookups by key), not ad hoc relational queries with joins.

### When to Use It
- High-scale, low-latency application backends: shopping carts, user sessions, user profiles, leaderboards, gaming state, IoT telemetry, ad tech, metadata stores.
- Workloads with massive or unpredictable traffic spikes where a relational database would struggle to scale writes.
- Serverless architectures (Lambda + API Gateway + DynamoDB) where connection management to a relational DB is a pain point.

### When *Not* to Use It
- Complex ad hoc queries, multi-table joins, or heavy aggregations → use RDS/Aurora (OLTP) or Redshift/Athena (analytics).
- Workloads whose access patterns are unknown or constantly changing — DynamoDB rewards upfront design and punishes guessing.
- Large binary objects (items are capped at **400 KB**) → store the object in S3 and keep a pointer in DynamoDB.

---

## 2. Core Concepts: Tables, Items, Attributes

| DynamoDB Term | Relational Analogy | Notes |
|---|---|---|
| **Table** | Table | A collection of items; no fixed schema beyond the key. |
| **Item** | Row | A single record, up to **400 KB** (including attribute names and values). |
| **Attribute** | Column | A name/value pair; can differ per item. |
| **Primary Key** | Primary key | Uniquely identifies each item; the **only** mandatory, schema-defined part. |

### Data Types
- **Scalar**: String, Number, Binary, Boolean, Null.
- **Document**: List and Map (nested, up to 32 levels deep) — lets you store JSON-like structures in a single attribute.
- **Set**: String Set, Number Set, Binary Set (unique, unordered values).

> **Interview tip:** Because attribute **names** count toward the 400 KB item limit, very long attribute names waste space and money at scale — short names are a real (if minor) optimization on huge tables.

---

## 3. Primary Keys

Every table must define a primary key, in one of two forms:

| Type | Composition | Behavior |
|---|---|---|
| **Simple primary key** | **Partition key** only | The partition key value is hashed to determine the physical partition; each value must be unique across the table. |
| **Composite primary key** | **Partition key + Sort key** | Items sharing a partition key are stored together, **ordered by sort key**; the *combination* must be unique. |

### Why the Sort Key Matters
- Enables **range queries** within a partition: `begins_with`, `between`, `<`, `>`, etc. — e.g., "all orders for customer 123 in the last 30 days."
- Lets you store a **one-to-many** relationship under a single partition key (one customer → many orders).
- The primary key (and its type) **cannot be changed** after table creation — changing it means creating a new table and migrating data.

### Choosing a Good Partition Key
- **High cardinality**: many distinct values (e.g., `user_id`, `order_id`), not `status` or `country`.
- **Uniform access**: requests spread evenly across key values rather than concentrating on a few "hot" keys ([Section 13](#13-hot-partitions-throttling--retries)).

---

## 4. Partitions & How Data Is Distributed

DynamoDB stores data in **partitions** — SSD-backed storage units managed entirely by the service. The partition key value is passed through an internal **hash function** to decide which partition holds the item.

### Key Facts
- Each partition holds roughly **10 GB** of data and supports up to **3,000 read capacity units (RCU)** and **1,000 write capacity units (WCU)** per second.
- DynamoDB **splits partitions automatically** as data volume or throughput grows — you never manage partitions directly.
- A table's total throughput is effectively spread across its partitions; a **single partition key value** can never exceed one partition's limits, no matter how much table-level capacity you provision.
- **Adaptive capacity** automatically shifts unused throughput to hotter partitions and can isolate heavily accessed items onto their own partitions, softening (but not eliminating) the impact of uneven access.

> **Interview tip:** "I provisioned 10,000 WCU but I'm still getting throttled" almost always means a **hot partition** — the table has capacity, but one key is exceeding the ~1,000 WCU single-partition ceiling.

---

## 5. Capacity Modes: Provisioned vs On-Demand

| Aspect | Provisioned | On-Demand |
|---|---|---|
| How you pay | Per RCU/WCU provisioned per hour, whether used or not | Per read/write request actually made |
| Capacity planning | Required — you set RCU/WCU (optionally with auto scaling) | None — scales automatically |
| Throttling risk | If traffic exceeds provisioned capacity (and burst) | Rare; can occur if traffic more than roughly doubles the previous peak in a very short window |
| Best for | Steady, predictable traffic; cost-optimized at high, stable utilization | New workloads, spiky or unpredictable traffic, "set it and forget it" |
| Cost at scale | Cheaper per request at consistent high utilization (and eligible for **reserved capacity** discounts) | Higher per-request price, but no waste on idle capacity |

### Details Worth Knowing
- **Auto scaling** (provisioned mode) adjusts RCU/WCU between a min/max based on a target utilization (commonly ~70%) — it reacts to *sustained* changes, not instantaneous spikes, so sharp spikes can still throttle.
- **Burst capacity**: DynamoDB retains up to ~5 minutes of unused provisioned capacity to absorb short bursts — don't design around it, treat it as a safety margin.
- You can **switch between modes**, but only **once every 24 hours** per table.
- **Table classes**: *Standard* (default) vs *Standard-Infrequent Access* (lower storage cost, higher read/write cost) — pick Standard-IA for tables that are large but rarely accessed (logs, history).

---

## 6. Calculating RCUs and WCUs

This is a favorite interview exercise — know the unit sizes cold.

| Unit | Definition |
|---|---|
| **1 RCU** | One **strongly consistent** read per second of an item up to **4 KB** — *or* two **eventually consistent** reads/sec — *or* half of one **transactional** read/sec |
| **1 WCU** | One write per second of an item up to **1 KB** |

- Item sizes are **rounded up** to the next 4 KB (reads) or 1 KB (writes).
- **Transactional** reads and writes cost **2×** their standard equivalents.

### Worked Examples

**Reads:** 10 reads/second of 6 KB items
- Strongly consistent: ⌈6/4⌉ = 2 RCU × 10 = **20 RCU**
- Eventually consistent: 20 ÷ 2 = **10 RCU**
- Transactional: 20 × 2 = **40 RCU**

**Writes:** 5 writes/second of 2.5 KB items
- Standard: ⌈2.5/1⌉ = 3 WCU × 5 = **15 WCU**
- Transactional: 15 × 2 = **30 WCU**

> **Interview tip:** A `Scan` or `Query` consumes RCUs based on the **total size of data read** (summed then rounded to 4 KB), **not** the number of items returned — which is why filters don't make a scan cheaper ([Section 8](#8-query-vs-scan)).

---

## 7. Read Consistency Models

| Model | Behavior | Cost | Notes |
|---|---|---|---|
| **Eventually consistent** (default) | May briefly return stale data (typically consistent within a second or so) | 0.5 RCU per 4 KB | Highest throughput, lowest cost. |
| **Strongly consistent** | Returns the most recent committed write | 1 RCU per 4 KB | Must be requested explicitly (`ConsistentRead=true`); higher latency; **not available on GSIs or in global table cross-Region reads**; unavailable if there's a network delay/outage in the leader's AZ. |
| **Transactional** | ACID, serializable isolation across items | 2 RCU per 4 KB | Used via `TransactGetItems`. |

---

## 8. Query vs Scan

| Aspect | Query | Scan |
|---|---|---|
| What it does | Retrieves items by **partition key** (optionally narrowed by a sort key condition) | Reads **every item** in the table or index |
| Efficiency | Reads only matching items in one partition — fast and cheap | Reads the entire table — slow and expensive, gets worse as the table grows |
| Typical use | The normal, intended access path | Rare: one-off analysis, small tables, exports (prefer **Export to S3**) |

### Important Behaviors
- **1 MB limit per call**: both return at most 1 MB of data; use `LastEvaluatedKey` for **pagination**.
- **Sort order**: a `Query` returns items ordered by sort key (ascending by default); set `ScanIndexForward=false` for descending — e.g., "newest orders first" with no extra index.
- **Filter expressions** are applied **after** items are read — you still **pay RCUs for all data read**, you just return less over the wire. A filter is not an index.
- **Projection expressions** return only chosen attributes (saves bandwidth, but RCUs are still based on full item size read).
- **Parallel Scan** splits a scan into segments processed by multiple workers — faster, but consumes capacity aggressively and can starve production traffic.
- **PartiQL** offers SQL-compatible syntax for the same operations — convenience, not a different engine; a PartiQL `SELECT` without a key condition is still a scan.

### Example (boto3)

```python
import boto3
from boto3.dynamodb.conditions import Key

table = boto3.resource("dynamodb").Table("Orders")

# Efficient: Query by partition key + sort key range
resp = table.query(
    KeyConditionExpression=Key("customer_id").eq("C123")
                         & Key("order_date").between("2026-01-01", "2026-01-31")
)
items = resp["Items"]

# Pagination
while "LastEvaluatedKey" in resp:
    resp = table.query(
        KeyConditionExpression=Key("customer_id").eq("C123"),
        ExclusiveStartKey=resp["LastEvaluatedKey"],
    )
    items += resp["Items"]
```

---

## 9. Secondary Indexes: LSI vs GSI

Secondary indexes let you query by attributes other than the table's primary key.

| Aspect | Local Secondary Index (LSI) | Global Secondary Index (GSI) |
|---|---|---|
| Partition key | **Same** as the base table | **Different** (any attribute) |
| Sort key | Different from the base table | Different (optional) |
| Creation | **Only at table creation** | Anytime, add/remove freely |
| Capacity | **Shares** the base table's capacity | Has **its own** provisioned/on-demand capacity |
| Consistency | Eventually **or strongly** consistent reads | **Eventually consistent only** |
| Size limit | 10 GB per partition key value (item collection limit) | None |
| Quota | 5 per table | 20 per table (default, adjustable) |
| Best for | Alternate sort order within the same partition | A completely different access pattern |

### Projection Types
Choose which attributes are copied into the index: `KEYS_ONLY`, `INCLUDE` (specific attributes), or `ALL`. Projecting fewer attributes lowers storage and write cost; querying for an attribute that isn't projected requires a costly fetch back to the base table (LSI) or isn't possible (GSI).

### GSI Gotchas (Frequently Tested)
- A GSI is **asynchronously** updated from the base table, so it is always eventually consistent.
- **Write back-pressure**: if a GSI doesn't have enough write capacity, writes to the **base table are throttled** too — under-provisioned GSIs hurt the whole table.
- **Sparse indexes**: an item only appears in a GSI if it has the index's key attributes — a deliberate trick to index just a subset (e.g., only items with `status = "PENDING"`).
- Each write that changes an indexed attribute costs additional writes (one per affected index).

---

## 10. Data Modeling & Single-Table Design

**DynamoDB modeling starts from access patterns, not entities.** List every query the application needs, then design keys and indexes to serve them.

### Modeling Process
1. Enumerate access patterns (e.g., "get user by ID," "list a user's orders newest-first," "get all orders in status X").
2. Choose the primary key to serve the most important/highest-volume pattern.
3. Add GSIs only for patterns the primary key can't serve.
4. Use **composite/hierarchical sort keys** to support range and prefix queries.

### Single-Table Design
Storing multiple entity types (users, orders, items) in **one table**, using generic key names (`PK`, `SK`) and prefixes to distinguish them:

| PK | SK | Attributes |
|---|---|---|
| `USER#123` | `PROFILE` | name, email |
| `USER#123` | `ORDER#2026-03-15#9001` | total, status |
| `USER#123` | `ORDER#2026-04-02#9017` | total, status |

One `Query` on `PK = USER#123` can return a user's profile **and** orders together; `begins_with(SK, "ORDER#")` returns only orders, already sorted by date.

### Common Patterns
- **Adjacency list**: model many-to-many relationships by storing edges as items.
- **Overloaded GSI**: reuse generic GSI key attributes (`GSI1PK`, `GSI1SK`) for different access patterns across entity types.
- **Write sharding**: append a random or calculated suffix to a hot partition key (`SCORE#7`) and fan out reads across suffixes.
- **Denormalization**: duplicate data to avoid joins — storage is cheap, reads that need multiple round trips are not.

### Single-Table vs Multi-Table

| Aspect | Single-table | Multi-table |
|---|---|---|
| Fewer round trips for related data | ✅ Yes | ❌ More calls |
| Readability / ease of change | ❌ Harder; patterns baked into keys | ✅ Simpler, closer to relational thinking |
| Per-entity settings (TTL, backup, scaling, streams) | ❌ Shared | ✅ Independent |
| Analytics/export friendliness | ❌ Mixed entities | ✅ Cleaner |

> **Interview tip:** Single-table design is a powerful technique, not a mandate. A strong answer acknowledges the trade-off: use it when you need to fetch related entities in one request at scale; prefer separate tables when entities have different lifecycles or when team velocity and clarity matter more than shaving a round trip.

---

## 11. Write Operations, Conditional Writes & Batching

### Core Operations
| Operation | Purpose |
|---|---|
| `PutItem` | Create or **fully replace** an item. |
| `UpdateItem` | Modify specific attributes (or create the item if absent — an *upsert*); supports atomic counters. |
| `DeleteItem` | Remove an item. |
| `GetItem` | Read one item by full primary key. |
| `BatchGetItem` | Read up to **100 items** (16 MB) across tables. |
| `BatchWriteItem` | Up to **25** put/delete requests (16 MB); **no updates**, **no conditions**, not atomic. |

> `BatchWriteItem` and `BatchGetItem` can return **`UnprocessedItems`/`UnprocessedKeys`** under throttling — your code must retry those with backoff.

### Conditional Writes
Make a write succeed only if a condition holds — evaluated atomically on the server:

```python
# Insert only if the item does not already exist (prevent overwrite)
table.put_item(
    Item={"PK": "USER#123", "SK": "PROFILE", "email": "a@b.com"},
    ConditionExpression="attribute_not_exists(PK)",
)
```

A failed condition raises `ConditionalCheckFailedException` (and still consumes write capacity).

### Atomic Counters
```python
table.update_item(
    Key={"PK": "POST#42", "SK": "STATS"},
    UpdateExpression="ADD view_count :inc",
    ExpressionAttributeValues={":inc": 1},
)
```
Atomic counters are not idempotent — a retried request double-counts; use a conditional write or transaction if exactness matters.

### Optimistic Locking
Store a `version` attribute; update with `ConditionExpression="version = :expected"` and increment it. A concurrent writer's stale version fails the condition, preventing lost updates. (The DynamoDB Enhanced Client for Java offers this natively via `@DynamoDbVersionAttribute`.)

---

## 12. Transactions

DynamoDB supports **ACID transactions** across multiple items — even across tables in the same Region and account.

| API | Description |
|---|---|
| `TransactWriteItems` | Up to **100 items** (4 MB total) of Put/Update/Delete/ConditionCheck — **all succeed or all fail**. |
| `TransactGetItems` | Up to **100 items** read together as a consistent snapshot. |

### Key Points
- **Cost**: each item in a transaction consumes **2×** the normal capacity (a prepare phase and a commit phase).
- **Idempotency**: supply a `ClientRequestToken`; retries with the same token within ~10 minutes won't re-apply the transaction.
- **ConditionCheck** lets a transaction depend on an item it doesn't modify (e.g., "only decrement stock if the product is still active").
- Transactions are **not supported across Regions** (including global table replicas) and conflict if two transactions touch the same item concurrently (`TransactionConflictException`).
- Typical use: money transfers, inventory reservation, enforcing uniqueness on a second attribute (write a "lock" item and the main item in one transaction).

---

## 13. Hot Partitions, Throttling & Retries

### Hot Partitions
A **hot partition/key** occurs when a disproportionate share of traffic targets one partition key value (a celebrity user, a single "current" date key, a global counter).

#### Mitigations
- Choose **high-cardinality** partition keys and avoid time-only keys for write-heavy data.
- **Write sharding**: add a suffix (`DATE#2026-10-26#3`) and spread writes across N shards; query all shards and merge on read.
- **Cache hot reads** with DAX ([Section 17](#17-dynamodb-accelerator-dax)) or ElastiCache.
- **Buffer spiky writes** through SQS/Kinesis and drain at a controlled rate.
- Use **Contributor Insights** to *find* the hot keys ([Section 21](#21-monitoring--observability)).

### Throttling and Errors
| Error | Meaning |
|---|---|
| `ProvisionedThroughputExceededException` | Request rate exceeded provisioned (or partition) capacity. |
| `ThrottlingException` | Control-plane or on-demand throttling. |
| `ConditionalCheckFailedException` | A condition expression evaluated to false (an expected application outcome, not a fault). |
| `TransactionConflictException` | Concurrent conflicting transaction on the same item. |

### Retry Strategy
- AWS SDKs retry throttled requests automatically using **exponential backoff with jitter** — keep this behavior, tune max retries rather than disabling it.
- Always retry **`UnprocessedItems`/`UnprocessedKeys`** from batch calls manually.
- If throttling persists, fix the cause (capacity, hot key, key design) rather than adding more retries.

---

## 14. DynamoDB Streams

A **stream** is an ordered, time-ordered log of item-level changes in a table (inserts, updates, deletes), retained for **24 hours**.

### Stream View Types
| View Type | Contents |
|---|---|
| `KEYS_ONLY` | Only the key attributes of the changed item |
| `NEW_IMAGE` | The entire item after the change |
| `OLD_IMAGE` | The entire item before the change |
| `NEW_AND_OLD_IMAGES` | Both — most flexible, enables diffing |

### Key Points
- **Ordering** is guaranteed **per item** (per partition key), not across the whole table.
- Delivery is **exactly-once into the stream**, but consumers should still be **idempotent** (Lambda retries can reprocess batches).
- The standard consumer is **AWS Lambda** via an event source mapping — see the Lambda notes on stream behavior (poison pill records, `BisectBatchOnFunctionError`, on-failure destinations).
- TTL deletions appear in the stream, flagged as service-initiated.

### Typical Use Cases
- Event-driven workflows (send a welcome email on user creation).
- Maintaining **materialized views/aggregates** in another table.
- **Replication** to Elasticsearch/OpenSearch, S3, or analytics stores.
- Audit trails and change data capture (CDC).

### DynamoDB Streams vs Kinesis Data Streams for DynamoDB

| Aspect | DynamoDB Streams | Kinesis Data Streams (for DynamoDB) |
|---|---|---|
| Retention | 24 hours | Up to 365 days |
| Consumers | Limited (best with ≤2 concurrent readers per shard) | Many, with enhanced fan-out |
| Ordering / duplicates | Per item, each change appears exactly once | **Not guaranteed**: records can arrive out of order and the same change can appear more than once — reconcile using the record's approximate creation timestamp |
| Downstream ecosystem | Lambda, KCL adapter | Firehose, Managed Flink, Kinesis analytics tooling |
| Best for | Simple event-driven triggers | Long retention, many consumers, streaming analytics |

---

## 15. Time to Live (TTL)

**TTL** automatically deletes expired items at **no write-capacity cost**.

### How It Works
- You designate a **Number attribute** holding a **Unix epoch timestamp (in seconds)**.
- A background process deletes items after that time — typically within **48 hours** of expiry (not instantly).
- Expired but not-yet-deleted items **still appear in reads**; filter on the TTL attribute in queries if exact expiry matters.
- Deletions are recorded in **Streams** (useful for archiving expired items to S3 via Lambda) and replicate through global tables.

### Use Cases
- Session data, temporary tokens/OTPs, cache entries, event/log retention windows, GDPR-style retention limits.

> **Interview tip:** TTL deletions are **free** (no WCU consumed), making TTL far cheaper than running a scheduled job that scans and deletes old items.

---

## 16. Global Tables

**Global tables** provide a fully managed, **multi-Region, multi-active** database — every replica accepts reads **and writes**, and changes replicate to all other Regions automatically.

### Key Points
- Current version is **2019.11.21**; a legacy 2017.11.29 version also exists. Provides a **99.999%** availability design target, low-latency local reads/writes worldwide, and built-in **multi-Region active-active** architecture.
- Replicas must share the same table name and key schema; capacity should be sized in every Region (on-demand mode or provisioned with auto scaling).
- Billing includes **replicated write request units** in each replica Region, so cost scales roughly with the number of Regions.

### Two Consistency Modes

| Aspect | MREC (multi-Region eventual consistency) — **default** | MRSC (multi-Region strong consistency) — GA since June 2025 |
|---|---|---|
| Replication | Asynchronous, typically **under a second** | Synchronous across Regions |
| RPO | Seconds (near zero, not zero) | **Zero** |
| Conflict resolution | **Last writer wins**, by timestamp — concurrent writes to the same item in different Regions resolve to the latest one | Strong consistency avoids conflicting divergent writes |
| Strongly consistent reads | Only guaranteed in the Region where the write happened; other Regions are eventually consistent | Strongly consistent reads return the latest data from **any** replica Region |
| Transactions | Atomic **within a Region** only (not preserved across replicas) | **Transaction APIs not supported** |
| Latency | Lowest write and strongly consistent read latency | Higher write latency (cross-Region coordination) |
| Limitations | Supports multi-account global tables | Same-account only, limited to specific Region sets (cannot span them), no `ReplicationLatency` metric |

> **Interview tip:** Default to MREC for most multi-Region apps; choose MRSC when the requirement is **RPO of zero** and always reading the latest data from any Region (e.g., inventory, financial records), accepting higher write latency and no transactions.

### Global Tables vs Cross-Region Read Replicas (RDS)
| Aspect | DynamoDB Global Tables | RDS Cross-Region Read Replica |
|---|---|---|
| Writes in multiple Regions | ✅ Yes (multi-active) | ❌ No (single writer; replica is read-only until promoted) |
| Replication | Asynchronous, typically sub-second | Asynchronous, can lag more |
| Failover | No promotion step — all replicas already writable | Manual promotion required |

---

## 17. DynamoDB Accelerator (DAX)

**DAX** is a fully managed, **in-memory write-through cache** purpose-built for DynamoDB, taking read latency from milliseconds to **microseconds**.

### Key Points
- **API-compatible** with DynamoDB — swap the client endpoint to the DAX cluster, with minimal code change.
- Runs as a **cluster inside your VPC** (one primary and read replica nodes across AZs).
- Maintains two caches: an **item cache** (`GetItem`/`BatchGetItem`) and a **query cache** (`Query`/`Scan` results), each with a default TTL of 5 minutes.
- **Write-through**: writes go through DAX to DynamoDB and update the item cache.
- Only **eventually consistent** reads are served from cache; **strongly consistent** and transactional reads **pass through** to DynamoDB.

### When to Use / Avoid
| Use DAX for | Avoid DAX for |
|---|---|
| Read-heavy, repeated reads of the same items (leaderboards, product catalogs) | Write-heavy workloads (little caching benefit) |
| Hot-key reads that risk throttling | Applications requiring strongly consistent reads for most requests |
| Microsecond latency requirements | Cases where an app-level cache (ElastiCache/Redis) with richer data structures is needed |

> **DAX vs ElastiCache**: DAX is DynamoDB-specific and transparent (no cache-management code); ElastiCache is general-purpose, but you write and manage the caching logic yourself.

---

## 18. Backup, Restore & Import/Export

### Backup Options
| Option | Details |
|---|---|
| **Point-in-Time Recovery (PITR)** | Continuous backups; restore to any second within a configurable window of **1–35 days**; no impact on table performance. |
| **On-demand backups** | Full backups you trigger (or schedule via **AWS Backup**), retained until deleted; support cross-Region copy and cross-account sharing via AWS Backup. |

### Key Points
- **Restores always create a new table** — there is no in-place restore; reconfigure application endpoints/table names afterward.
- Some settings (such as auto scaling policies, TTL, streams, alarms, and PITR settings) generally need to be **re-applied** on the restored table — verify against current docs.
- Backups consume **no read capacity** and don't affect table performance.
- Turn on **deletion protection** for production tables to block accidental `DeleteTable` calls.

### Export to and Import from S3
- **Export to S3** (from PITR data): produces DynamoDB JSON or Amazon Ion files for analytics with Athena/Glue/EMR — **consumes no table capacity**, making it the right alternative to a full `Scan`.
- **Import from S3**: creates a **new table** from CSV, DynamoDB JSON, or Ion data in S3 — avoids writing a loader and consuming WCUs.

---

## 19. Security

### Encryption
- **Encryption at rest** is **always on** — choose the key type: **AWS owned key** (default, free), **AWS managed key** (`aws/dynamodb` in KMS), or **customer managed KMS key** (full control, audit, rotation).
- **In transit**: all API calls use **HTTPS/TLS**.

### Access Control
- **IAM identity-based policies** govern API access (`dynamodb:GetItem`, `Query`, etc.); **resource-based policies** can also be attached to tables/streams for cross-account sharing.
- **Fine-grained access control** using IAM condition keys:
  - `dynamodb:LeadingKeys` — restrict access to items whose partition key matches a value (e.g., a user's own ID via `${cognito-identity.amazonaws.com:sub}`) — effectively **row-level security**.
  - `dynamodb:Attributes` — restrict which attributes can be read/written (column-level).

### Network
- Use a **Gateway VPC Endpoint** for DynamoDB so traffic from private subnets reaches the service without a NAT Gateway or the internet — it's **free** (see the VPC notes on gateway endpoints).
- DynamoDB is a regional public service by design; there is no "put it in a subnet" setting like RDS.

### Auditing
- **CloudTrail** logs control-plane (and optionally data-plane) API activity; enable data events for compliance-sensitive tables.

---

## 20. Integrations with Other AWS Services

| Service | Integration |
|---|---|
| **AWS Lambda** | Triggered by DynamoDB Streams; also the typical compute layer reading/writing tables. |
| **API Gateway** | Direct service integration to expose table operations as APIs without Lambda. |
| **AppSync** | GraphQL resolvers map directly to DynamoDB operations. |
| **Kinesis Data Streams / Firehose** | Stream item changes for long retention or delivery to S3/OpenSearch. |
| **S3** | Export/import of table data; archive TTL-expired items. |
| **Athena / Glue / EMR** | Analytics over exported data (or federated queries via connectors). |
| **Redshift / OpenSearch Service** | **Zero-ETL integrations** replicate DynamoDB data for analytics and search without a custom pipeline. |
| **Step Functions** | Native task integrations for reading/writing items in workflows. |
| **AWS Backup** | Centralized, policy-based scheduled backups and retention. |

> **Interview tip:** Because DynamoDB can't do ad hoc analytics or full-text search, the standard pattern is **DynamoDB for operational access + a purpose-built store (Redshift/Athena for analytics, OpenSearch for search) fed via Streams, Export to S3, or Zero-ETL**.

---

## 21. Monitoring & Observability

### Key CloudWatch Metrics
| Metric | Why It Matters |
|---|---|
| `ConsumedReadCapacityUnits` / `ConsumedWriteCapacityUnits` | Actual usage vs provisioned — drives auto scaling and right-sizing. |
| `ReadThrottleEvents` / `WriteThrottleEvents` / `ThrottledRequests` | Direct signal of capacity or hot-partition problems. |
| `SystemErrors` / `UserErrors` | 5xx service errors vs 4xx client errors. |
| `SuccessfulRequestLatency` | Per-operation latency trends. |
| `ReplicationLatency` | Lag between global table replicas (published for MREC global tables only, not MRSC). |
| `IteratorAge` (on the consuming Lambda) | How far behind your Streams consumer is. |

### Contributor Insights
- Surfaces the **most accessed and most throttled partition keys** — the go-to tool to find hot keys.

### Alarms Worth Setting
- Throttle events > 0 sustained, consumed capacity approaching provisioned (for non-auto-scaled tables), replication latency spikes, and stream consumer lag.

---

## 22. DynamoDB vs Other Databases

| Aspect | DynamoDB | RDS / Aurora | DocumentDB | Keyspaces (Cassandra) | ElastiCache |
|---|---|---|---|---|---|
| Model | Key-value / document | Relational (SQL) | Document (MongoDB-compatible) | Wide-column (Cassandra-compatible) | In-memory key-value |
| Scaling | Horizontally, virtually unlimited, automatic | Vertical + read replicas (Aurora scales further) | Cluster with replicas | Serverless, horizontally | Cluster/sharding |
| Query flexibility | Limited to key-based patterns + indexes | Full SQL, joins, ad hoc | Rich document queries, aggregation | CQL, partition-key-driven | Simple key lookups/data structures |
| Transactions | Yes (limited scope) | Full ACID | Multi-document ACID | Lightweight transactions only | Limited |
| Latency | Single-digit ms | Low ms | Low ms | Single-digit ms | Sub-millisecond |
| Operations burden | None (serverless) | Instance/cluster management | Cluster management | None (serverless) | Cluster management |
| Best for | Known access patterns, massive scale, serverless | Relational data, complex queries | Existing MongoDB apps | Existing Cassandra apps | Caching, sessions, real-time counters |

> **Interview tip:** "Why DynamoDB over RDS?" — scale without capacity planning for known access patterns and zero operations. "Why RDS over DynamoDB?" — relational integrity, ad hoc queries, joins, and evolving query needs.

---

## 23. Pricing & Cost Optimization

### Cost Components
- **Read/write throughput**: provisioned RCU/WCU-hours, or per-request charges in on-demand mode.
- **Storage**: per GB-month (lower for Standard-IA table class).
- **Optional features**: PITR (per GB), on-demand backups, global table replicated writes, DAX nodes, Streams reads beyond free tier, Kinesis integration, data transfer out.

### Cost Levers
1. **Right-size the capacity mode**: on-demand for spiky/unknown traffic; provisioned + auto scaling (and **reserved capacity**) for steady traffic.
2. **Use eventually consistent reads** wherever acceptable — half the read cost.
3. **Avoid Scans**; design keys and GSIs so access is `Query`/`GetItem`.
4. **Project only needed attributes** into GSIs and keep items small (short attribute names, offload large blobs to S3).
5. **Use TTL** to delete expired data for free instead of paying to store it or delete it manually.
6. **Standard-IA table class** for large, rarely accessed tables.
7. **Batch** where it reduces request overhead, and **cache** hot reads (DAX/ElastiCache) to cut read charges.
8. **Delete unused GSIs** — each indexed write is an extra write cost.

---

## 24. Limits Quick Reference

| Limit | Value |
|---|---|
| Max item size | 400 KB |
| Partition key length | 1–2,048 bytes |
| Sort key length | 1–1,024 bytes |
| Query/Scan response page | 1 MB |
| `BatchGetItem` | 100 items / 16 MB |
| `BatchWriteItem` | 25 requests / 16 MB |
| Transaction | 100 items / 4 MB |
| Per-partition throughput | 3,000 RCU / 1,000 WCU |
| LSIs per table | 5 (created at table creation only) |
| GSIs per table | 20 (default quota) |
| LSI item collection size | 10 GB per partition key value |
| Streams retention | 24 hours |
| PITR window | 1–35 days |
| Capacity mode switch | Once per 24 hours |

(Quotas can change and some are adjustable — confirm against current AWS documentation before relying on exact numbers.)

---

## 25. Best Practices

1. **Design for access patterns first** — list every query before choosing keys ([Section 10](#10-data-modeling--single-table-design)).
2. **Pick high-cardinality partition keys** with even access distribution to avoid hot partitions ([Sections 3, 13](#3-primary-keys)).
3. **Prefer `Query`/`GetItem` over `Scan`**; use Export to S3 for bulk analytics ([Section 8](#8-query-vs-scan)).
4. **Keep items small**; store large objects in S3 and reference them by key ([Section 1](#1-what-is-dynamodb)).
5. **Use GSIs deliberately** — size their write capacity properly and project only what you need ([Section 9](#9-secondary-indexes-lsi-vs-gsi)).
6. **Use conditional writes and optimistic locking** for correctness under concurrency instead of read-modify-write in application code ([Section 11](#11-write-operations-conditional-writes--batching)).
7. **Make stream consumers idempotent** and configure retries/failure destinations ([Section 14](#14-dynamodb-streams)).
8. **Use TTL for expiring data** — it's free and removes cleanup jobs ([Section 15](#15-time-to-live-ttl)).
9. **Enable PITR on production tables** and use AWS Backup for scheduled/long-term retention ([Section 18](#18-backup-restore--importexport)).
10. **Use a Gateway VPC Endpoint** for private-subnet access instead of routing through a NAT Gateway ([Section 19](#19-security)).
11. **Apply least-privilege IAM**, using `LeadingKeys` for per-user row-level access where appropriate ([Section 19](#19-security)).
12. **Monitor throttling and use Contributor Insights** to catch hot keys early ([Section 21](#21-monitoring--observability)).
13. **Pick the capacity mode deliberately** and revisit it as traffic matures ([Section 5](#5-capacity-modes-provisioned-vs-on-demand)).

---

## 26. Common Interview Questions

1. **What are the two types of primary keys in DynamoDB?** → Simple (partition key) and composite (partition + sort key) ([Section 3](#3-primary-keys)).
2. **How is data distributed across partitions, and what limits does a single partition have?** → Hash of the partition key; ~10 GB, 3,000 RCU / 1,000 WCU per partition ([Section 4](#4-partitions--how-data-is-distributed)).
3. **Provisioned vs on-demand capacity — how do you choose?** → Steady/predictable (provisioned + auto scaling, reserved capacity) vs spiky/unknown (on-demand) ([Section 5](#5-capacity-modes-provisioned-vs-on-demand)).
4. **How many RCUs for 10 strongly consistent reads/sec of 6 KB items?** → ⌈6/4⌉ × 10 = 20 RCU (10 if eventually consistent, 40 if transactional) ([Section 6](#6-calculating-rcus-and-wcus)).
5. **Why do I get throttled even though I have unused table capacity?** → A hot partition exceeding per-partition limits ([Sections 4, 13](#4-partitions--how-data-is-distributed)).
6. **Query vs Scan — and does a filter expression reduce the cost of a scan?** → Query uses the key; Scan reads everything; filters apply after reading, so RCUs are still consumed on all data read ([Section 8](#8-query-vs-scan)).
7. **LSI vs GSI — differences and when would you pick each?** → LSI: same partition key, creation-time only, shares capacity, supports strong consistency; GSI: any keys, anytime, own capacity, eventually consistent ([Section 9](#9-secondary-indexes-lsi-vs-gsi)).
8. **How can an under-provisioned GSI affect the base table?** → Write throttling propagates back to the base table ([Section 9](#9-secondary-indexes-lsi-vs-gsi)).
9. **What is single-table design and what are its trade-offs?** → Multiple entity types in one table via generic keys, minimizing round trips at the cost of complexity and shared settings ([Section 10](#10-data-modeling--single-table-design)).
10. **How do you prevent overwriting an existing item?** → `ConditionExpression="attribute_not_exists(PK)"` on `PutItem` ([Section 11](#11-write-operations-conditional-writes--batching)).
11. **How do you handle concurrent updates safely?** → Conditional writes with a version attribute (optimistic locking) ([Section 11](#11-write-operations-conditional-writes--batching)).
12. **How do DynamoDB transactions work, and what do they cost?** → ACID across up to 100 items, 2× capacity per item, idempotency token for retries ([Section 12](#12-transactions)).
13. **How do you mitigate a hot partition key?** → High-cardinality keys, write sharding, caching, buffering writes, Contributor Insights to find it ([Section 13](#13-hot-partitions-throttling--retries)).
14. **DynamoDB Streams vs Kinesis Data Streams for DynamoDB?** → 24-hour retention/few consumers vs up to 365 days/many consumers ([Section 14](#14-dynamodb-streams)).
15. **How does TTL work and what does it cost?** → Epoch-seconds attribute; deletion within ~48 hours; free (no WCU); expired items may still be readable until deleted ([Section 15](#15-time-to-live-ttl)).
16. **How do global tables resolve write conflicts?** → Last writer wins by timestamp; transactions remain Region-local ([Section 16](#16-global-tables)).
17. **When would you use DAX versus ElastiCache?** → DAX for transparent, microsecond caching of DynamoDB reads (eventually consistent); ElastiCache for general-purpose or richer caching logic ([Section 17](#17-dynamodb-accelerator-dax)).
18. **How do you restore a table to an earlier point in time?** → PITR restore to a *new* table, then re-point the application and reapply settings ([Section 18](#18-backup-restore--importexport)).
19. **How do you give each user access to only their own items?** → IAM `dynamodb:LeadingKeys` condition on the partition key ([Section 19](#19-security)).
20. **How would you run analytics or full-text search over DynamoDB data?** → Export to S3 + Athena, Zero-ETL to Redshift, or Streams/Zero-ETL to OpenSearch ([Section 20](#20-integrations-with-other-aws-services)).
21. **How do you reach DynamoDB from a private subnet without a NAT Gateway?** → Gateway VPC Endpoint ([Section 19](#19-security)).
22. **DynamoDB vs RDS — when would you choose each?** → Known access patterns, extreme scale, zero ops vs relational integrity, joins, ad hoc queries ([Section 22](#22-dynamodb-vs-other-databases)).
