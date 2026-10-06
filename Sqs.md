## Table of Contents

1. [What is Amazon SQS?](#1-what-is-amazon-sqs)
2. [Standard Queues vs FIFO Queues](#2-standard-queues-vs-fifo-queues)
3. [Core Message Lifecycle](#3-core-message-lifecycle)
4. [Visibility Timeout](#4-visibility-timeout)
5. [Short Polling vs Long Polling](#5-short-polling-vs-long-polling)
6. [Dead-Letter Queues (DLQs)](#6-dead-letter-queues-dlqs)
7. [Message Attributes & Payload Size](#7-message-attributes--payload-size)
8. [Delay Queues & Message Timers](#8-delay-queues--message-timers)
9. [FIFO Deep Dive: Ordering & Deduplication](#9-fifo-deep-dive-ordering--deduplication)
10. [Encryption](#10-encryption)
11. [Access Control & Queue Policies](#11-access-control--queue-policies)
12. [SQS with Lambda (Event Source Mapping)](#12-sqs-with-lambda-event-source-mapping)
13. [SQS with SNS: Fan-Out Pattern](#13-sqs-with-sns-fan-out-pattern)
14. [Batching: Send, Receive, Delete](#14-batching-send-receive-delete)
15. [Monitoring & CloudWatch Metrics](#15-monitoring--cloudwatch-metrics)
16. [Scaling & Throughput](#16-scaling--throughput)
17. [Idempotency & Exactly-Once-ish Processing](#17-idempotency--exactly-once-ish-processing)
18. [SQS vs SNS vs Kinesis vs EventBridge](#18-sqs-vs-sns-vs-kinesis-vs-eventbridge)
19. [Pricing](#19-pricing)
20. [Best Practices](#20-best-practices)
21. [Common Interview Questions](#21-common-interview-questions)

---

## 1. What is Amazon SQS?

**Amazon Simple Queue Service (SQS)** is a fully managed **message queuing service** that lets you decouple and scale microservices, distributed systems, and serverless applications by passing messages between producers and consumers through a durable, highly available queue.

### Why It Exists
Without a queue, a producer calling a consumer directly creates **tight coupling**: if the consumer is slow, down, or overwhelmed, the producer either blocks or fails. SQS breaks that coupling — producers drop messages into the queue and move on; consumers pull messages whenever they're ready, at their own pace.

### Key Characteristics
- **Fully managed**: no servers to provision, patch, or scale — AWS handles all of it.
- **Durable**: messages are redundantly stored across multiple servers and Availability Zones.
- **Pull-based**: unlike SNS or EventBridge (push-based), consumers actively **poll** SQS for messages — there is no "push to consumer" mechanism built into SQS itself.
- **At-least-once delivery** (Standard queues) — a message may occasionally be delivered more than once, so consumers should be idempotent ([Section 17](#17-idempotency--exactly-once-ish-processing)).
- **Retention**: messages are retained for a configurable period, from **60 seconds up to 14 days** (default 4 days), after which they're automatically deleted if unprocessed.

### Common Use Cases
- **Decoupling microservices** — a web app enqueues work (e.g., order processing) for a backend worker fleet to consume asynchronously.
- **Buffering/load leveling** — absorbing a traffic spike so a downstream system (e.g., a database with limited write capacity) isn't overwhelmed.
- **Work distribution** — multiple consumer instances pulling from the same queue to parallelize processing of a large job queue.
- **Decoupling Lambda from a bursty event source** — smoothing out invocation concurrency instead of letting every event trigger an immediate, simultaneous Lambda invocation.

---

## 2. Standard Queues vs FIFO Queues

SQS offers two queue types, with fundamentally different ordering and delivery guarantees.

| Aspect | Standard Queue | FIFO Queue |
|---|---|---|
| Ordering | **Best-effort** — order is not guaranteed | **Strict order** preserved within a message group |
| Delivery | **At-least-once** — duplicates possible | **Exactly-once processing** (with deduplication) — no duplicates introduced by SQS itself |
| Throughput | Nearly unlimited | Up to 3,000 msg/sec per API call with batching (or 300/sec without batching); higher with batching across multiple message groups |
| Naming | Any name | Must end in `.fifo` |
| Use case | High-throughput, order-insensitive workloads (logging, metrics ingestion, general task queues) | Order-sensitive workloads (financial transactions, sequential commands, e-commerce order processing) |
| Cost | Slightly cheaper per request | Slightly more expensive per request |

> **Interview tip:** "Best-effort ordering" on Standard queues means messages are *usually* delivered in the order sent, but **not guaranteed** — and a message might be delivered **more than once**. If your workload cannot tolerate either of those, you need a FIFO queue.

---

## 3. Core Message Lifecycle

1. **SendMessage** — a producer sends a message to the queue; SQS durably stores it.
2. **ReceiveMessage** — a consumer polls the queue and retrieves one or more messages (up to 10 per call). The message becomes **invisible** to other consumers for the duration of the **visibility timeout** ([Section 4](#4-visibility-timeout)), but is **not yet deleted**.
3. **Process** — the consumer does its work with the message.
4. **DeleteMessage** — once processing succeeds, the consumer explicitly deletes the message from the queue using its **receipt handle**. If the consumer never deletes it (crash, timeout, bug), the message becomes visible again after the visibility timeout expires and is **redelivered** to another consumer.

### Key Point
- A message is only truly "gone" after an explicit `DeleteMessage` call — SQS has no concept of "the consumer finished, so remove it automatically." This is what makes at-least-once delivery possible: if the consumer dies mid-processing, the undeleted message simply reappears for someone else to try.

---

## 4. Visibility Timeout

The **visibility timeout** is the period during which a message that has been received by one consumer is **hidden from other consumers**, giving that consumer time to process and delete it.

### Key Points
- Default: **30 seconds**; configurable from **0 seconds up to 12 hours**, per queue or per individual `ReceiveMessage`/`ChangeMessageVisibility` call.
- If the consumer doesn't delete the message before the timeout expires, it becomes visible again and can be picked up (and processed) by another consumer — this is the primary mechanism behind SQS's at-least-once guarantee and its handling of consumer failures.
- If your processing time is unpredictable or can run long, call **`ChangeMessageVisibility`** mid-processing to extend the timeout dynamically, instead of guessing a single large value up front.

### Setting It Too Short vs Too Long

| Setting | Risk |
|---|---|
| **Too short** | The message becomes visible again *before* the consumer finishes — a second consumer may start processing the same message concurrently, risking duplicate side effects. |
| **Too long** | If a consumer crashes right after receiving a message, that message is effectively "stuck" (invisible to everyone) for the full timeout duration before it's retried — slowing down recovery from failures. |

> **Interview tip:** Visibility timeout should generally be set to **at least 6x your function's/consumer's typical processing time** when SQS is the trigger for Lambda — AWS's own guidance for the Lambda event source mapping, to avoid premature redelivery while a Lambda invocation (and its retries) are still in flight.

---

## 5. Short Polling vs Long Polling

### Short Polling (default historically)
- `ReceiveMessage` returns **immediately**, even if no messages are available, or returns only a subset of messages from a sample of SQS's backend servers.
- Can result in **empty responses** even when messages exist elsewhere in the distributed queue backend, and wastes API calls (and cost) when the queue is often empty.

### Long Polling (recommended)
- `ReceiveMessage` **waits** (up to 20 seconds, via `WaitTimeSeconds`) for a message to arrive before returning, instead of returning empty immediately.
- Queries **all** backend servers, so it reliably returns a message if one exists anywhere in the queue.
- Reduces the number of empty responses, cutting the **number of billable requests** and reducing latency in picking up new messages.

### Comparison

| Aspect | Short Polling | Long Polling |
|---|---|---|
| `WaitTimeSeconds` | 0 | 1–20 seconds |
| Empty responses | Common, even with messages present | Rare — waits for a message or timeout |
| Cost efficiency | Lower (more wasted empty calls) | Higher (fewer wasted calls) |
| Recommended | ❌ Legacy default | ✅ Yes, virtually always |

---

## 6. Dead-Letter Queues (DLQs)

A **Dead-Letter Queue** is a regular SQS queue designated as the destination for messages that **repeatedly fail processing**, so they don't block the main queue or get silently lost.

### How It Works
1. Configure a **redrive policy** on the source queue, specifying a target DLQ and a `maxReceiveCount`.
2. Each time a message is received without being deleted (i.e., it times out and becomes visible again), SQS increments its **receive count**.
3. Once `maxReceiveCount` is exceeded, SQS automatically moves the message to the DLQ instead of redelivering it to the source queue again.

### Why Use a DLQ
- Prevents a single malformed or "poison pill" message from being retried **forever**, endlessly consuming consumer capacity.
- Isolates problem messages for **manual inspection and debugging**, separate from the healthy message flow.
- A **redrive capability** lets you move messages back from the DLQ to the source queue (e.g., after fixing the bug that caused failures) without writing custom code.

### Key Points
- A DLQ is just a **normal SQS queue** — it has no special type; what makes it a "DLQ" is that another queue's redrive policy points to it.
- The DLQ should generally be the **same type** as its source queue (Standard DLQ for a Standard source queue, FIFO DLQ for a FIFO source queue).
- Set a CloudWatch alarm on the DLQ's `ApproximateNumberOfMessagesVisible` metric — an empty DLQ with sudden messages appearing is often the first signal of a new processing bug.

---

## 7. Message Attributes & Payload Size

### Message Size Limits
- Maximum message size: **256 KB** (body + attributes combined).
- For larger payloads, use the **SQS Extended Client Library**, which transparently stores the actual payload in **S3** and sends only a small pointer/reference message through SQS — a common pattern for passing large files (e.g., a video to transcode) through a queue-based pipeline.

### Message Attributes
- Up to **10 message attributes** per message — structured key-value metadata (e.g., `ContentType: String = "application/json"`) sent alongside the message body.
- Useful for routing/filtering logic in consumers without needing to parse the full message body first.
- Distinct from the message body — attributes count toward the 256 KB size limit but are retrieved and processed separately.

### Message Attributes vs Message Body

| Aspect | Message Body | Message Attributes |
|---|---|---|
| Purpose | The actual payload/content | Structured metadata about the message |
| Format | Free-form text/JSON/XML | Typed key-value pairs (String, Number, Binary) |
| Max count | N/A (one body) | 10 attributes |
| Used by | Consumer business logic | Routing, filtering (e.g., SNS-to-SQS filter policies), tracing |

---

## 8. Delay Queues & Message Timers

### Delay Queues
- A **queue-level** setting that delays the delivery of **all** new messages sent to the queue by a fixed amount, from **0 seconds up to 15 minutes**.
- Useful when you always want a buffer period before any message in the queue becomes visible — e.g., giving a related system time to finish a prerequisite step.

### Message Timers
- A **per-message** delay (also 0 seconds to 15 minutes) set on an individual `SendMessage` call via the `DelaySeconds` parameter, overriding the queue's default delay for that specific message.
- Useful for scheduling an individual message's visibility independently of the rest of the queue (e.g., "retry this specific message in 5 minutes").

### Key Point
- **Not supported on FIFO queues** at the per-message level in the same way — message timers on FIFO queues are more limited; check current documentation, as this is a frequently-tested nuance. Delay queues (queue-level) do work on FIFO queues.

---

## 9. FIFO Deep Dive: Ordering & Deduplication

### Message Groups
- Every message in a FIFO queue belongs to a **message group** (set via `MessageGroupId`).
- **Strict ordering is guaranteed only within a single message group** — messages across different message groups may be processed in parallel and interleaved, with no ordering guarantee between groups.
- Using a **single message group ID** for everything guarantees global ordering but limits throughput to one consumer effectively processing in sequence; using **many message group IDs** (e.g., one per customer or order ID) allows parallelism while still guaranteeing order *within* each entity's own messages.

### Deduplication
FIFO queues prevent duplicate messages from being **introduced into the queue** (note: this is different from a consumer accidentally processing a successfully-delivered message twice, which idempotent design still protects against). Two deduplication mechanisms:

| Mechanism | How It Works |
|---|---|
| **Content-based deduplication** | SQS computes a SHA-256 hash of the message body automatically; if an identical message is sent again within the **5-minute deduplication interval**, it's treated as a duplicate and not enqueued again. |
| **Explicit deduplication ID** | The producer sets a `MessageDeduplicationId` directly — useful when two messages might have identical bodies but should *not* be deduplicated, or vice versa (e.g., deduplicating based on a business transaction ID rather than the literal payload). |

### Exactly-Once Processing
- FIFO's "exactly-once processing" claim refers to SQS not **introducing** duplicates into the queue, combined with it removing a message once successfully deleted — it is **not** an absolute guarantee that a consumer's side effects can never happen twice (e.g., a consumer could process a message, perform a side effect, then crash before calling `DeleteMessage`, causing redelivery). Consumer-side idempotency is still a best practice even with FIFO ([Section 17](#17-idempotency--exactly-once-ish-processing)).

---

## 10. Encryption

### Encryption in Transit
- All API calls to SQS use **HTTPS/TLS** by default.

### Encryption at Rest (SSE)

| Type | Who Manages the Key | Notes |
|---|---|---|
| **SSE-SQS** | AWS manages the key (Amazon-owned) | Default, simplest option, no extra cost, enabled automatically for new queues as of recent AWS defaults. |
| **SSE-KMS** | AWS KMS-managed customer key (CMK) | Adds auditability (CloudTrail logs key usage) and fine-grained key policies; incurs standard KMS request costs and is subject to KMS request-rate quotas at very high throughput. |

### Key Point
- With SSE-KMS, be mindful of **KMS request quotas** — a high-throughput queue making a KMS call for every send/receive can hit KMS throttling limits; AWS mitigates this with **data key caching** inside SQS, but it's worth knowing as a potential bottleneck in a very high-volume design discussion.

---

## 11. Access Control & Queue Policies

### IAM Policies
- Standard identity-based IAM policies control which principals can call SQS actions (`sqs:SendMessage`, `sqs:ReceiveMessage`, `sqs:DeleteMessage`, etc.) on specific queue ARNs — the primary mechanism for same-account access control.

### Queue Policies (Resource-Based)
- A **resource-based policy** attached directly to the queue, similar in structure to an S3 bucket policy — used primarily for:
  - **Cross-account access**: granting another AWS account's principals permission to send/receive without needing a role in your account.
  - **Allowing another AWS service to send messages** — e.g., an S3 bucket's event notifications, or an SNS topic subscription, need the queue policy to explicitly allow that service principal to call `sqs:SendMessage`.

### Example: Allowing SNS to Publish to a Queue
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "sns.amazonaws.com" },
      "Action": "sqs:SendMessage",
      "Resource": "arn:aws:sqs:us-east-1:123456789012:my-queue",
      "Condition": {
        "ArnEquals": { "aws:SourceArn": "arn:aws:sns:us-east-1:123456789012:my-topic" }
      }
    }
  ]
}
```

---

## 12. SQS with Lambda (Event Source Mapping)

SQS is one of the most common Lambda triggers, using the **poll-based Event Source Mapping (ESM)** pattern — Lambda's own internal poller reads from the queue and invokes your function synchronously with a batch of messages.

### Key Configuration
- **Batch size**: up to 10,000 messages per batch (for Standard queues; lower effective limits for FIFO), or capped by a **batch window** (max time to wait for a batch to fill before invoking anyway).
- **Concurrency**: Lambda scales the number of pollers based on queue depth, up to the function's concurrency limit (and any Reserved Concurrency cap).
- **Partial batch failure reporting**: Lambda can report back *which specific messages* in a batch failed, so only those are returned to the queue for redelivery — instead of the default behavior where a single thrown error causes the **entire batch** to be retried, even messages that succeeded.

### Visibility Timeout Guidance
- Set the queue's visibility timeout to **at least 6x the function's timeout** — if the function is still running (or retrying) when the message becomes visible again, a second concurrent invocation could process the same message.

### SQS as a Lambda DLQ Target
- Don't confuse this with [Section 6](#6-dead-letter-queues-dlqs)'s SQS-native DLQs — Lambda **itself** can also route failed *asynchronous* invocations to an SQS queue as an on-failure destination. When SQS is the **trigger** (poll-based), failure handling instead uses the event source mapping's own redrive policy pointing to a DLQ, as described in Section 6.

---

## 13. SQS with SNS: Fan-Out Pattern

A very common architecture: an **SNS topic** publishes a single message to **multiple SQS queue subscribers simultaneously** — each queue gets its own independent copy, consumed by a different downstream service at its own pace.

### Why This Pattern
- SNS alone is push-based with no built-in retry/durability if a subscriber is down; combining it with SQS adds **durability and buffering** per-subscriber — if one consumer service is down for an hour, its queue simply holds the messages until it recovers, without affecting the other subscribers.
- Lets you add a **new consumer** to an existing event stream just by subscribing a new SQS queue to the topic, with zero changes to the producer or other consumers.

### Example Flow
```
Producer → SNS Topic → ┬→ SQS Queue A → Consumer A (e.g., billing service)
                        ├→ SQS Queue B → Consumer B (e.g., analytics service)
                        └→ SQS Queue C → Consumer C (e.g., notification service)
```

### SNS Filter Policies
- Each SQS subscription to the SNS topic can have a **filter policy**, so a given queue only receives the subset of messages matching specific message attributes — avoiding the need for every consumer to receive (and then discard) every message.

---

## 14. Batching: Send, Receive, Delete

Batching reduces the number of billable API requests and improves throughput by operating on up to **10 messages per call**.

| API | Purpose |
|---|---|
| `SendMessageBatch` | Send up to 10 messages in a single API call. |
| `ReceiveMessage` | Already supports retrieving up to 10 messages per call via `MaxNumberOfMessages`. |
| `DeleteMessageBatch` | Delete up to 10 messages in a single API call, using their receipt handles. |
| `ChangeMessageVisibilityBatch` | Extend/change the visibility timeout for up to 10 messages at once. |

### Key Point
- Batching is a significant **cost optimization**: SQS bills per request, and a batch of 10 messages counts as far fewer billable requests than sending/deleting them one at a time — always prefer batch APIs in high-throughput producers/consumers.

---

## 15. Monitoring & CloudWatch Metrics

| Metric | Meaning |
|---|---|
| `ApproximateNumberOfMessagesVisible` | Messages currently available to be received — the primary "queue depth" indicator. |
| `ApproximateNumberOfMessagesNotVisible` | Messages currently "in flight" (received but not yet deleted or timed out). |
| `ApproximateAgeOfOldestMessage` | How long the oldest unprocessed message has been sitting in the queue — a rising value signals consumers can't keep up. |
| `NumberOfMessagesSent` / `NumberOfMessagesReceived` / `NumberOfMessagesDeleted` | Throughput counters for the three core operations. |
| `NumberOfEmptyReceives` | Count of `ReceiveMessage` calls that returned no messages — a high count signals short polling is being used inefficiently. |

### Common Alarms
- Alarm on `ApproximateAgeOfOldestMessage` exceeding a threshold — a strong signal of consumer-side backpressure or an outage, often better than alarming on raw queue depth alone (depth can legitimately be high on a healthy, high-throughput queue).
- Alarm on the DLQ's `ApproximateNumberOfMessagesVisible` going above zero — signals new messages are failing processing ([Section 6](#6-dead-letter-queues-dlqs)).

---

## 16. Scaling & Throughput

### Standard Queues
- **Virtually unlimited throughput** — SQS scales transparently in the background; there's no number of messages/second you need to provision for.

### FIFO Queues
- Without batching: up to **300 messages/second** (send, receive, or delete combined) per API action.
- With batching (up to 10 messages per batch call): up to **3,000 messages/second**.
- **High throughput mode** (default for new FIFO queues) removes the need to shard across multiple message group IDs purely for throughput reasons — it scales FIFO throughput much higher as long as you're using many distinct message group IDs, since per-group-ID throughput is still bounded.

### Consumer-Side Scaling
- SQS itself doesn't limit how many consumers can poll a queue concurrently — throughput from the **consumer side** scales by adding more polling workers (EC2 instances, ECS tasks, or Lambda's automatically-scaled pollers).

---

## 17. Idempotency & Exactly-Once-ish Processing

Because **Standard queues guarantee at-least-once delivery** (and even FIFO's "exactly-once" has the caveats noted in [Section 9](#9-fifo-deep-dive-ordering--deduplication)), consumers should always be written to be **idempotent**: processing the same message twice produces the same end result as processing it once.

### Common Idempotency Patterns
- Use a **unique identifier** from the message (or generate one) as a key in a datastore (e.g., a DynamoDB table) with a **conditional write** — if the key already exists, skip reprocessing.
- Design downstream operations to be naturally idempotent where possible — e.g., `SET balance = 100` instead of `balance += 10` (the former is safe to repeat; the latter is not).
- For Lambda consumers specifically, the **Lambda Powertools idempotency utility** provides a ready-made implementation of this pattern.

---

## 18. SQS vs SNS vs Kinesis vs EventBridge

A classic "which AWS messaging service would you use" interview comparison.

| Aspect | SQS | SNS | Kinesis Data Streams | EventBridge |
|---|---|---|---|---|
| Model | Pull-based queue | Push-based pub/sub | Pull-based stream (ordered, replayable) | Push-based event bus with routing rules |
| Delivery | At-least-once (Standard) / exactly-once-ish (FIFO) | At-least-once, multiple protocols (SQS, Lambda, HTTP, email, SMS) | At-least-once, ordered per shard, replayable | At-least-once, rule-based routing to many targets |
| Message retention | Up to 14 days | None (fire-and-forget; no retention if no subscriber) | Up to 365 days (configurable) | None (events are routed immediately, not stored) |
| Ordering | Best-effort (Standard) / strict per group (FIFO) | None | Strict per shard | None |
| Replay | ❌ No | ❌ No | ✅ Yes (re-read from any point in the retention window) | ❌ No (native), though can archive+replay as a separate feature |
| Fan-out | Via SNS (SQS is 1 queue : many consumers pulling the same queue, not natively multi-subscriber) | ✅ Native (one topic, many subscriber types) | ✅ Native (many consumers can read the same stream independently) | ✅ Native (one event, many rule-matched targets) |
| Best for | Decoupling point-to-point work, buffering, task queues | Fan-out notifications to multiple heterogeneous subscribers | High-throughput ordered streaming/analytics pipelines, replay needs | SaaS/third-party event routing, complex content-based routing across many AWS services |

> **Interview tip:** A common framing — **SQS decouples a producer from consumers pulling work at their own pace; SNS broadcasts to many subscribers at once; Kinesis is for ordered, replayable streaming data at scale; EventBridge is for routing events (including from SaaS/third-party sources) based on content to many different targets.** They're frequently combined (e.g., SNS → multiple SQS queues for durable fan-out, [Section 13](#13-sqs-with-sns-fan-out-pattern)).

---

## 19. Pricing

- Billed per **request** (a request is any API action — `SendMessage`, `ReceiveMessage`, `DeleteMessage`, etc.), with a monthly free tier.
- **Batched requests count as a single request** regardless of how many messages are in the batch (up to 10) — a major cost lever for high-volume producers/consumers ([Section 14](#14-batching-send-receive-delete)).
- FIFO queue requests are billed at a slightly higher rate than Standard queue requests.
- No charge for data storage while messages sit in the queue — cost is purely a function of API request volume.
- SSE-KMS encryption adds standard KMS request pricing on top of SQS's own request pricing.

---

## 20. Best Practices

1. **Always use long polling** (`WaitTimeSeconds` 1–20) instead of short polling to reduce cost and empty responses ([Section 5](#5-short-polling-vs-long-polling)).
2. **Always configure a DLQ** with a sensible `maxReceiveCount` — never let a poison-pill message retry forever ([Section 6](#6-dead-letter-queues-dlqs)).
3. **Set visibility timeout to ~6x your consumer's expected processing time**, and extend it dynamically for long-running work instead of guessing one large static value ([Section 4](#4-visibility-timeout)).
4. **Design consumers to be idempotent** — assume at-least-once delivery even on FIFO queues ([Section 17](#17-idempotency--exactly-once-ish-processing)).
5. **Use batch APIs** (`SendMessageBatch`, `DeleteMessageBatch`) wherever producer/consumer throughput allows, for both cost and latency benefits ([Section 14](#14-batching-send-receive-delete)).
6. **Use the Extended Client Library (S3-backed payloads)** instead of splitting or compressing data to fit under the 256 KB message limit ([Section 7](#7-message-attributes--payload-size)).
7. **Choose FIFO only when you genuinely need strict ordering or no-duplicate-introduction** — Standard queues are cheaper and scale further; don't default to FIFO "just in case" ([Section 2](#2-standard-queues-vs-fifo-queues)).
8. **Use many message group IDs on FIFO queues** when you need both ordering *and* throughput — a single group ID serializes everything ([Section 9](#9-fifo-deep-dive-ordering--deduplication)).
9. **Monitor `ApproximateAgeOfOldestMessage`**, not just queue depth, to detect genuine consumer backpressure ([Section 15](#15-monitoring--cloudwatch-metrics)).
10. **Use SNS + SQS fan-out** instead of having a single producer call multiple downstream services directly — adds durability and makes adding new consumers a zero-touch change for existing ones ([Section 13](#13-sqs-with-sns-fan-out-pattern)).

---

## 21. FAQs

1. **Standard vs FIFO — when would you choose each?** → [Section 2](#2-standard-queues-vs-fifo-queues) — ordering/duplicate sensitivity vs. raw throughput and lower cost.
2. **How does SQS know a message was successfully processed?** → It doesn't, implicitly — the consumer must explicitly call `DeleteMessage`; otherwise the message reappears after the visibility timeout ([Sections 3–4](#3-core-message-lifecycle)).
3. **What happens if a consumer crashes mid-processing?** → The message becomes visible again after the visibility timeout expires and is redelivered to another consumer — this is the basis of at-least-once delivery ([Section 4](#4-visibility-timeout)).
4. **How do you prevent a bad message from blocking a queue forever?** → A Dead-Letter Queue with a `maxReceiveCount` redrive policy ([Section 6](#6-dead-letter-queues-dlqs)).
5. **Why prefer long polling over short polling?** → Reduces empty responses and billable requests, and more reliably returns an available message ([Section 5](#5-short-polling-vs-long-polling)).
6. **How do you send a 50 MB file through SQS, given the 256 KB limit?** → The SQS Extended Client Library, storing the payload in S3 and passing a reference through the queue ([Section 7](#7-message-attributes--payload-size)).
7. **How does FIFO guarantee order while still allowing some parallelism?** → Strict order is guaranteed only within a message group; using multiple message group IDs allows parallel processing across groups while preserving per-group order ([Section 9](#9-fifo-deep-dive-ordering--deduplication)).
8. **SQS vs SNS — what's the core difference?** → Pull-based point-to-point queue vs. push-based pub/sub fan-out; often combined ([Sections 13, 18](#18-sqs-vs-sns-vs-kinesis-vs-eventbridge)).
9. **SQS vs Kinesis — when would you pick Kinesis instead?** → When you need message replay, very high ordered throughput per partition, or multiple independent consumers reading the same stream at their own position ([Section 18](#18-sqs-vs-sns-vs-kinesis-vs-eventbridge)).
10. **How do you connect SQS to Lambda, and what's the risk if you don't set visibility timeout correctly?** → Event Source Mapping polling; too-short a timeout risks duplicate concurrent processing of the same message while the function is still running ([Section 12](#12-sqs-with-lambda-event-source-mapping)).
11. **Does FIFO guarantee a consumer will never process the same message twice?** → No — it guarantees SQS won't *enqueue* duplicates, but a consumer that crashes after processing but before deleting the message can still see it again; idempotent consumer design is still required ([Section 17](#17-idempotency--exactly-once-ish-processing)).
12. **How do you let an S3 bucket's event notifications deliver into an SQS queue?** → A queue policy granting the S3 service principal `sqs:SendMessage`, scoped with a `SourceArn` condition ([Section 11](#11-access-control--queue-policies)).
