# kinesis
Repo on aws kinesis 

---

# Amazon Kinesis Data Streams — Q&A 

**What is Amazon Kinesis Data Streams, and how does it work?**
**Answer:** Amazon Kinesis Data Streams (KDS) is a fully managed, real-time streaming service for ingesting, buffering, and processing large streams of event data. Producers write records to a stream. A stream is divided into shards (units of capacity). Consumers read records from shards (using iterators). Kinesis replicates data across AZs, stores events for a configurable retention window, and provides low-latency streaming for real-time analytics, ETL, and event-driven apps.

---

**What are the Kinesis family components and how do they differ?**
**Answer:**

* **Kinesis Data Streams (KDS)** — sharded stream for real-time ingest and custom consumers.
* **Kinesis Data Firehose** — fully managed delivery to S3/Redshift/ES; handles batching, compression, and retries (no custom consumers needed).
* **Kinesis Data Analytics** — runs SQL/Flink on streams for real-time transformations and windowed aggregations.
* **Kinesis Video Streams** — for streaming video (different domain).
  Choose KDS when you need custom, low-latency consumers and fine-grained control.

---

**What is a shard? What are its throughput limits?**
**Answer:** A shard is the unit of capacity in KDS. Per shard limits: **write** up to **1 MB/sec OR 1,000 records/sec**; **read** (classic) **2 MB/sec** shared among consumers; with **Enhanced Fan-Out (EFO)** each consumer gets **2 MB/sec** per shard. If you exceed these, you get provisioned throughput exceptions.

---

**What is a partition key and how does it affect ordering?**
**Answer:** Partition key is a string supplied by the producer. Kinesis hashes it to a 128-bit hash value and maps that value to a shard’s hash key range. All records with the same partition key go to the same shard — this preserves ordering for that key. Ordering is guaranteed **within a single shard only** (no global ordering across shards).

---

**How long does Kinesis retain data?**
**Answer:** Default retention is **24 hours**. It can be configured up to **7 days** for standard retention (and longer with extended retention/streams in some AWS offerings). Older data beyond retention is expired and removed.

---

**What are the typical consumer options?**
**Answer:**

* **Lambda trigger** — easiest to deploy; AWS polls and invokes Lambda with batches. Good for simple processing.
* **KCL (Kinesis Client Library)** — handles leases, checkpointing (DynamoDB), worker coordination; ideal for heavy, long-running processing.
* **Enhanced Fan-Out (EFO) / SubscribeToShard** — push-based, low-latency per-consumer throughput.
* **Kinesis Data Analytics (Flink)** — stateful streaming with windowing and joins.

---

**How do you produce records efficiently (best practices)?**
**Answer:** Use **PutRecords** (batch API) or **KPL (Kinesis Producer Library)** to aggregate small events into fewer records. Keep records ≤1 MB and batches ≤500 records per PutRecords call. Also ensure partition keys distribute load evenly.

**Node.js PutRecords example (AWS SDK v3):**

```js
import { KinesisClient, PutRecordsCommand } from "@aws-sdk/client-kinesis";
const k = new KinesisClient({ region: "us-east-1" });
await k.send(new PutRecordsCommand({
  StreamName: "mystream",
  Records: [
    { Data: Buffer.from(JSON.stringify({ x:1 })), PartitionKey: "user-1" },
    { Data: Buffer.from(JSON.stringify({ x:2 })), PartitionKey: "user-2" }
  ]
}));
```

---

**How does Kinesis ensure durability?**
**Answer:** When Kinesis acknowledges a PutRecord/PutRecords, it has synchronously replicated the record across multiple AZs to durable storage. Records are kept for the configured retention window and are addressable by sequence number. For consumer state durability, KCL stores checkpoints in DynamoDB.

---

**What is an iterator and what are iterator types?**
**Answer:** A shard iterator specifies where a GetRecords call should start reading. Types: `TRIM_HORIZON` (oldest available), `LATEST` (only new incoming records), `AT_SEQUENCE_NUMBER`, `AFTER_SEQUENCE_NUMBER`, `AT_TIMESTAMP`.

---

**What is checkpointing and who is responsible?**
**Answer:** Checkpointing records the last processed sequence number from a shard so consumers can resume after restart. **KCL** handles checkpointing using DynamoDB. If you implement your own consumer, you must persist checkpoints (e.g., DynamoDB) to prevent reprocessing or data loss.

---

**How do you handle duplicates and implement idempotency?**
**Answer:** Kinesis is **at-least-once**, so consumers must be idempotent. Strategies: include `eventId` in events and store dedupe keys in a dedupe store (DynamoDB with conditional writes & TTL), use idempotent upserts, or dedupe on downstream systems. See dedupe Node.js example from earlier snippets.

---

**What is a hot shard and how do you mitigate it?**
**Answer:** A hot shard occurs when a shard receives most of the traffic (often due to skewed partition keys), hitting its write/read limit. Mitigations:

* Add entropy to partition keys (e.g., `userId#bucket`)
* Use composite partition keys or hashing/bucketing
* Split the hot shard (targeted SplitShard)
* Move heavy producers to a dedicated stream
* Use KPL aggregation to reduce per-second record rate

---

**Explain shard split and merge at a high level.**
**Answer:**

* **Split**: one shard’s hash key range is split into two child shards. The parent shard closes for new writes; child shards take new writes for subranges. Consumers will read remaining parent data then new child data.
* **Merge**: two adjacent shards with contiguous hash ranges are merged into one shard. Used to reduce shard count when load decreases. Both operations keep sequence numbers consistent per shard and are coordinated by Kinesis control plane.

---

**How do you programmatically split a hot shard?**
**Answer:** Use the Kinesis API `SplitShard` (provide `ShardToSplit` and `NewStartingHashKey`) or call `UpdateShardCount` for uniform scaling. Targeted splits require you to compute a midpoint hash key for the shard’s hash key range. (See earlier `splitShard.js` snippet — compute midpoint and call `SplitShardCommand`.)

---

**How do you autoscale shards?**
**Answer:** Monitor CloudWatch metrics (`IncomingBytes`, `IncomingRecords`, `WriteProvisionedThroughputExceeded`, `IteratorAgeMilliseconds`). Use a Lambda autoscaler or AWS Application Auto Scaling to call `UpdateShardCount` with `UNIFORM_SCALING` when metrics cross thresholds. Apply cooldowns and caps to avoid oscillation. (See earlier `autoscaler-lambda.js` snippet.)

---

**How can you support multiple consumers with different SLAs (dashboards vs. ETL)?**
**Answer:** Use **Enhanced Fan-Out** for low-latency, high-concurrency consumers (dashboards, online inference) and classic consumers (KCL/Lambda) for ETL/offline jobs. Or use Firehose to deliver to S3 for heavy ETL consumers.

---

**What metrics and alarms should you monitor for Kinesis?**
**Answer:**

* `IncomingBytes` / `IncomingRecords` (ingest load)
* `WriteProvisionedThroughputExceeded` / `ReadProvisionedThroughputExceeded` (throttling)
* `GetRecords.IteratorAgeMilliseconds` (consumer lag)
* `PutRecord.Success` / `PutRecords.FailedRecordCount` (producer errors)
* `ProvisionedThroughputExceeded` spikes -> alert & scale

---

**What are typical use cases for Kinesis?**
**Answer:** Real-time analytics (clickstream), metrics ingestion, real-time ETL, streaming ML inference, log/event collection, IoT telemetry, and fan-out to multiple downstream systems.

---

**What are Kinesis record size limits and how do you handle larger payloads?**
**Answer:** Single record max size: **1 MB**. For larger payloads, store the payload in S3 and put a pointer (S3 URL/key) in the Kinesis record. Alternatively compress/serialize the payload (Avro/Protobuf) to reduce size.

---

**How are security and encryption handled?**
**Answer:** Use IAM policies for least privilege on producers/consumers. Enable server-side encryption (SSE-KMS) for data at rest. Use TLS for data in transit and VPC endpoints (PrivateLink) for private network access.

---

**What is Enhanced Fan-Out (EFO)? When should you use it?**
**Answer:** EFO gives each consumer a dedicated 2 MB/sec per shard and push-based delivery via HTTP/2, lowering read latency (≈70 ms). Use EFO when multiple consumers require low-latency, high-throughput access (dashboards, real-time inference). EFO has additional cost per consumer-shard.

---

**How does Kinesis compare to Kafka (short bullet points)?**
**Answer:** Kinesis is managed (AWS handles replication and ops), has per-shard throughput limits, and uses HTTP APIs; Kafka (self-managed/MSK) gives more control (partitions, replication factor, transactions, log compaction), typically lower latency and richer client features but higher ops overhead.

---

**How do you achieve exactly-once processing semantics?**
**Answer:** Kinesis is at-least-once. For effective exactly-once, implement idempotent sinks and dedupe (use eventId + DynamoDB conditional writes), or implement transactional semantics at the sink where possible. Use strict checkpointing discipline with KCL combined with idempotent downstream writes.

---

**What is the max number of records per PutRecords and how do you handle failures?**
**Answer:** `PutRecords` can contain up to **500** records per call and total payload must obey per-record and request limits. On partial failures, the response includes per-record error codes — retry failed records with backoff or send failures to a DLQ.

---

**Interview-style operational questions (examples to prepare):**

* *What was your PutRecords batch size and why?*
  **Suggested answer:** I used 200–500 records per PutRecords call (bounded by latency/aggregate size), tuned so each batch was ≈500KB–1MB and to keep API calls efficient while limiting retry blast on partial failures.

* *How did you detect and mitigate hot shards in production?*
  **Suggested answer:** I monitored per-shard `IncomingBytes` and `WriteProvisionedThroughputExceeded`. For hot shards I added entropy to partition keys for new producers, performed targeted `SplitShard` on hot shard(s), and moved exceptionally heavy producers to a dedicated stream if needed.

* *What DLQ strategy did you use for failed consumer processing?*
  **Suggested answer:** I sent records that failed processing after N retries to an SQS DLQ (with full event metadata and error context) and set up alerting and a reprocessing pipeline for manual inspection/replay.

* *How did you size shards for expected throughput?*
  **Suggested answer:** Calculate both record/sec and bytes/sec needs. Example: 100k events/sec at 1KB each requires ~100 shards (record/sec limit mainly), so provisioned 120 shards for headroom and used autoscaling to adjust.

---

**Short Node.js consumer sample (Lambda-style processing with idempotent DynamoDB claim):**

```js
import { DynamoDBClient, PutItemCommand } from "@aws-sdk/client-dynamodb";
const ddb = new DynamoDBClient({ region: "us-east-1" });

export const handler = async (event) => {
  for (const r of event.Records) {
    const payload = Buffer.from(r.kinesis.data, "base64").toString("utf8");
    const obj = JSON.parse(payload);
    const eventId = obj.eventId;
    try {
      // Claim dedupe entry
      await ddb.send(new PutItemCommand({
        TableName: "dedupe",
        Item: { eventId: { S: eventId }, ts: { N: String(Date.now()) } },
        ConditionExpression: "attribute_not_exists(eventId)"
      }));
      // process obj...
    } catch (e) {
      if (e.name === "ConditionalCheckFailedException") {
        console.log("duplicate, skip", eventId);
      } else {
        throw e;
      }
    }
  }
};
```



# **📘 Amazon Kinesis Data Streams — Interview Questions & Answers**

---

### **What is Amazon Kinesis Data Streams, and how does it work?**

**Answer:**
Amazon Kinesis Data Streams (KDS) is a fully managed, real-time streaming service that ingests large volumes of data from producers (apps, microservices, IoT, clickstreams), stores them across *shards*, and allows multiple consumers to read and process the data simultaneously. It ensures low-latency, durable, ordered event delivery within each shard.

---

### **What are shards in Kinesis, and why are they important?**

**Answer:**
Shards are the fundamental unit of capacity in a Kinesis stream. Each shard supports:

* **Write:** 1,000 records/sec OR 1 MB/sec
* **Read:** 2 MB/sec shared (classic), OR 2 MB/sec per consumer (EFO)
  Shards determine scalability and parallelism. More shards = higher throughput.

---

### **What is a partition key, and how does it impact data distribution?**

**Answer:**
The partition key determines **which shard** a record goes to. Kinesis hashes the key (MD5) → maps it to a shard’s hash-key range.

* Same partition key = guaranteed ordering (within that shard)
* Bad or non-distributed partition keys = hot shards

---

### **Does Kinesis guarantee message ordering?**

**Answer:**
**Yes — but only within a shard.**
Records inside a single shard maintain strict ordering via sequence numbers.
Kinesis does *not* guarantee global ordering across multiple shards.

---

### **How does Kinesis provide durability?**

**Answer:**
Kinesis synchronously replicates records across **three Availability Zones** before acknowledging the PutRecord/PutRecords request. This ensures no single-AZ failure can cause data loss.

---

### **What is the retention period of a Kinesis stream?**

**Answer:**
Default: **24 hours**
Configurable: **up to 7 days** (standard)
Extended retention (long-term): available (additional cost).

---

### **What are Enhanced Fan-Out (EFO) consumers?**

**Answer:**
EFO provides each consumer its **own 2 MB/sec throughput per shard** and **70ms latency** using HTTP/2 push delivery.
Use EFO when:

* Multiple consumers
* Low-latency dashboards
* ML inference, monitoring, or real-time UIs

---

### **What are the common consumer types in Kinesis?**

**Answer:**

* **Lambda consumer** — simple serverless processing
* **KCL (Kinesis Client Library)** — advanced processing, checkpointing, scaling
* **Enhanced Fan-Out (EFO)** — low-latency push-based consumers
* **Kinesis Data Analytics (Flink SQL)** — windowing, aggregation, joins

---

### **Explain checkpointing in Kinesis Consumer applications.**

**Answer:**
Checkpointing tracks the last-processed sequence number in a shard.
KCL stores checkpoints in **DynamoDB**.
Benefits:

* Ensures progress
* Enables restart without reprocessing
* Coordinates shards across multiple workers

---

### **What is meant by a “hot shard”? Why does it happen?**

**Answer:**
A hot shard receives disproportionately high traffic → exceeding 1 MB/sec or 1,000 req/sec.
Common cause: badly distributed partition keys (e.g., all traffic = `"user123"`).

---

### **How do you fix uneven partition distribution or a hot shard?**

**Answer:**

* Add randomness to partition keys (`userId#bucket`)
* Use KPL (producer-side aggregation)
* Split the hot shard (`SplitShard`)
* Spread heavy producers across multiple keys
* Switch to on-demand stream mode (auto scaling)
* Re-partition with composite keys (`region-userId`)

---

### **Explain shard splitting and merging.**

**Answer:**

* **SplitShard** → 1 shard → 2 shards
  Used when traffic increases or shard becomes hot.
* **MergeShard** → 2 shards → 1 shard
  Used to reduce cost when traffic drops.
  After a split/merge, consumers must process parent shards before child shards (KCL handles this automatically).

---

### **What happens when a PutRecord call exceeds throughput?**

**Answer:**
Kinesis returns `ProvisionedThroughputExceededException`.
Producer must:

* Retry with exponential backoff
* Use batching
* Reduce request rate
* Fix partition keys
* Increase shard count

---

### **How do you calculate the required number of shards?**

**Answer:**
You must satisfy **both** of these:

1. **Based on records/sec:**

```
shards = ceil(total_records_per_sec / 1000)
```

2. **Based on MB/sec:**

```
shards = ceil(total_MB_per_sec / 1MB)
```

Choose the **larger** of the two.

---

### **How do you handle millions of messages per minute with Kinesis?**

**Answer:**

* Provision enough shards
* Use KPL aggregation
* Use Enhanced Fan-Out
* Auto-scale via CloudWatch metrics and `UpdateShardCount`
* Use Flink/KDS Analytics for windowed aggregates
* Apply backpressure on producers
* Place producers and consumers in same region (no cross-region)

---

### **What is KPL (Kinesis Producer Library) and why is it used?**

**Answer:**
KPL aggregates many small records into larger ones to reduce API calls and maximize throughput. It also does retries, time-based flushing, and efficient batching.

---

### **What are common use cases of Kinesis Data Streams?**

**Answer:**

* Clickstream analytics
* IoT telemetry
* Real-time fraud detection
* Payment/event streaming
* Log ingestion
* Multi-consumer fan-out pipelines
* ETL for ML systems (feature pipelines)

---

### **What is the maximum record size and how do you handle larger data?**

**Answer:**
Max record size = **1 MB**
To handle larger data:

* Upload to S3 and send **pointer/key** in Kinesis record
* Compress records before sending
* Split large events into smaller ones

---

### **What is the difference between Kinesis Data Streams and Firehose?**

**Answer:**

| Feature       | Kinesis Data Streams | Firehose                            |
| ------------- | -------------------- | ----------------------------------- |
| Consumer type | Custom consumers     | Auto-delivery                       |
| Latency       | 70–200ms             | 1–120 seconds                       |
| Control       | Full control         | Fully-managed                       |
| Use case      | Real-time processing | Streaming to S3/Redshift/OpenSearch |

---

### **How does KCL (Kinesis Client Library) work internally?**

**Answer:**
KCL uses DynamoDB to store:

* Lease ownership per shard
* Checkpoints
* Worker heartbeats

It automatically:

* Balances shards among workers
* Handles split/merge
* Recovers from worker failures
* Ensures exactly-once checkpointing (not exactly-once processing)

---

### **How do you implement idempotency for Kinesis consumers?**

**Answer:**
Because Kinesis is at-least-once:

* Include `eventId` in each message
* Write to DynamoDB with conditional check:
  `attribute_not_exists(eventId)`
* Deduplicate with TTL-based dedupe table
* Use idempotent sinks (upsert writes)

---

### **How do you scale Kinesis automatically?**

**Answer:**
Use a Lambda autoscaler to read CloudWatch metrics (`IncomingBytes`, `IncomingRecords`) and call `UpdateShardCount` to scale up/down.
Or use **Kinesis On-Demand mode** (automatic scaling by AWS).

---

### **How do you secure a Kinesis stream?**

**Answer:**

* IAM least-privilege policies
* Server-Side Encryption (SSE-KMS)
* VPC endpoints (Privatelink)
* Access policies for specific producers/consumers
* KMS key rotation if needed

---

### **What metrics help detect consumer lag?**

**Answer:**

* **GetRecords.IteratorAgeMilliseconds**
  If this increases → consumers are falling behind → increase consumer concurrency, use EFO, improve code speed.

---

### **How do you send multiple events in one request?**

**Answer:**
Use `PutRecords` (max 500 records per API call).

---

### **Node.js PutRecords Example:**

```js
import { KinesisClient, PutRecordsCommand } from "@aws-sdk/client-kinesis";

const client = new KinesisClient({ region: "us-east-1" });

await client.send(new PutRecordsCommand({
  StreamName: "orders-stream",
  Records: [
    { Data: Buffer.from("event-1"), PartitionKey: "user-1" },
    { Data: Buffer.from("event-2"), PartitionKey: "user-2" }
  ]
}));
```

---

### **Node.js Consumer Example (Lambda)**

```js
export const handler = async (event) => {
  for (const rec of event.Records) {
    const json = JSON.parse(Buffer.from(rec.kinesis.data,'base64').toString());
    console.log('Processing event:', json);
  }
};
```

---

# **Additional Interview Questions (SDE-3)**

### **What was your shard count and why?**

Explain calculations based on records/sec and MB/sec, plus future load estimates.

---

### **Did you use Extended Fan-Out? Why or why not?**

Use if multiple consumers with low-latency requirements. Skip if cost-sensitive.

---

### **How did you handle failures in consumer processing?**

Use DLQ (SQS) for failed events + CloudWatch alarms.

---

### **How did you debug consumer processing lag?**

* Check `IteratorAgeMilliseconds`
* Check KCL leasing
* Check DB bottleneck downstream
* Increase EFO throughput
* Add consumer parallelism

---

### **What was your retention window and why?**

E.g., 48 hours → supports reprocessing + analytics backlog safety.

---

### **Did you use KPL? Why?**

Use KPL when producer throughput is high and aggregation/batching is needed.

---

Below is the **same clean Q&A format** you used for SQS/SNS and Kinesis Data Streams — now fully written for **Amazon Kinesis Data Firehose**.
Short, crisp, interview-ready, and product-company friendly.

You can copy-paste this directly into your repo.

---

# **📘 Amazon Kinesis Data Firehose — Interview Questions & Answers**

---

### **What is Amazon Kinesis Data Firehose, and how does it work?**

**Answer:**
Amazon Kinesis Data Firehose is a fully managed service used to **load streaming data into AWS destinations** such as S3, Redshift, OpenSearch, and Splunk. Firehose automatically batches, compresses, transforms, and reliably delivers data without requiring custom consumers. It scales automatically and requires **zero management**.

---

### **How does Firehose differ from Kinesis Data Streams?**

**Answer:**

| Feature    | Kinesis Data Streams      | Kinesis Data Firehose                      |
| ---------- | ------------------------- | ------------------------------------------ |
| Consumer   | Custom (Lambda, KCL, EFO) | Fully automated                            |
| Latency    | 70–200ms                  | 1–120 seconds                              |
| Processing | Your code                 | Firehose does batching, compression, retry |
| Scaling    | Manual / On-demand        | Auto-scaling                               |
| Use cases  | Real-time apps            | Delivering to S3/Redshift/ES/Splunk        |

Firehose focuses on **managed delivery**, not custom streaming logic.

---

### **What destinations does Firehose support?**

**Answer:**

* **Amazon S3** (any format)
* **Amazon Redshift**
* **Amazon OpenSearch Service**
* **Splunk**
* **Custom HTTP endpoints**
* **Datadog, NewRelic, MongoDB Atlas** via partner destinations

---

### **Does Firehose require managing shards?**

**Answer:**
**No.** Firehose has **no shards**. It automatically scales based on incoming throughput.

---

### **What is Firehose’s delivery latency?**

**Answer:**

* **S3**: 1–120 seconds (default 60 seconds or 1MB buffer)
* **Redshift**: Firehose loads data into S3, then COPYs into Redshift → ~1–5 min
* **OpenSearch**: ~1 second to 60 seconds depending on buffer settings

---

### **What are Firehose buffer hints?**

**Answer:**
Buffer hints define when Firehose should flush data to the destination.
They include:

* **Buffer Size (1–128 MB)**
* **Buffer Interval (60–900 seconds)**

Firehose flushes **whichever limit is reached first**.

---

### **How does Firehose ensure reliability?**

**Answer:**

* Automatically retries failed deliveries
* Stores failed events temporarily
* Can route permanently failed events to **S3 Backup / DLQ Bucket**
* Supports at-least-once delivery
* Uses S3 as intermediary before loading to Redshift

---

### **What is backup (S3 Backup Mode) in Firehose?**

**Answer:**
Backup Mode sends:

* **All records** (primary + failed) OR
* **Only failed records**

to an S3 backup bucket so you can inspect or reprocess them.

---

### **Can Firehose transform data before delivering?**

**Answer:**
**Yes**, using:

1. **Lambda Transformations** — custom JS/Python code
2. **Record format conversion** — Parquet/ORC for S3
3. **Dynamic partitioning** based on record attributes (S3 partition folders)

---

### **What is Dynamic Partitioning in Firehose?**

**Answer:**
Dynamic partitioning allows Firehose to write to **different S3 prefixes** based on attribute values inside each event.
Example:
`/year=2025/month=01/day=15/hour=12/region=us-east-1/`

This is extremely useful for analytics (Athena, Glue, Redshift).

---

### **How does Firehose handle oversized records?**

**Answer:**
Max record size is **1 MB** (same as Kinesis Streams).
If larger:

* It can send the record to the **backup S3 bucket**
* Or reject it entirely

For huge records: upload to S3 → send pointer via Firehose.

---

### **What is the retry policy for destinations like Redshift or OpenSearch?**

**Answer:**
Firehose retries for:

* **OpenSearch**: up to 5 minutes by default
* **Redshift**: retries COPY commands
* For continuous failures → fail to S3 backup bucket

---

### **What is the difference between Firehose Direct Put and Kinesis Stream Source?**

**Answer:**
Firehose can receive data in two ways:

1. **Direct Put** — producer writes directly to Firehose
2. **Kinesis Stream Source** — Firehose reads from a Kinesis Data Stream

Use direct put when simple ingestion.
Use KDS source when you need custom processing first.

---

### **Explain how Firehose loads into Redshift.**

**Answer:**

1. Firehose stores records in an intermediate S3 bucket.
2. Firehose uses the `COPY` command to load data into Redshift in batches.
3. If the load fails, records are sent to backup S3.
4. Redshift rejects invalid rows → stored in S3 as well.

---

### **What compression formats does Firehose support?**

**Answer:**

* GZIP
* ZIP
* Snappy
* Hadoop-compatible compression
* Parquet/ORC (when using format conversion)

---

### **What happens when Firehose cannot reach its destination?**

**Answer:**
Firehose retries for a period, and if still failing:

* Stores data in S3 backup (if enabled)
* Sends CloudWatch alarms
* Continues retrying until recovered

Firehose never drops data silently.

---

### **How does Firehose scale automatically?**

**Answer:**
Firehose adjusts ingestion, batching, and delivery capacity based on load automatically.
No shard provisioning, no manual scaling, no throttling logic.

---

### **What are typical Firehose monitoring metrics?**

**Answer:**

* `DeliveryToS3.Success`
* `DeliveryToRedshift.Success`
* `BackupFailure`
* `BytesDelivered`
* `DeliveryToOpenSearch.*`
* `ThrottledRecords`
* `KinesisMillisBehindLatest` (when using KDS source)

---

### **How do you handle schema evolution in Firehose?**

**Answer:**

* Use Parquet/ORC conversion for schema tracking
* Use Lambda transformation to modify schema
* Store raw events in S3 backup for reprocessing

---

### **What happens if Lambda transformation fails?**

**Answer:**
Firehose follows this flow:

* Retries lambda
* If still failing → sends record to **Backup S3**
* Marks transformation as failed in CloudWatch

---

### **Can Firehose encrypt data?**

**Answer:**
Yes:

* Encryption at rest using KMS
* In-transit via TLS
* S3 objects can be encrypted (SSE-S3 or SSE-KMS)

---

### **What is a common Firehose architecture?**

**Answer:**
Producers → Firehose → (Optional Lambda transform) → S3/Redshift/OpenSearch
Plus Backup S3 for DLQ events.

---

### **Can Firehose push to multiple destinations?**

**Answer:**
No. One Firehose delivery stream = one primary destination.
But you can chain Firehose + Lambda → multiple Firehoses or Streams.

---

### **Firehose vs Kafka + Connect?**

**Answer:**
Firehose is AWS-native and fully managed with zero operations.
Kafka Connect is more flexible but requires more setup, maintenance, workers, connectors, scaling, etc.

---

### **Typical Firehose use cases:**

* Log aggregation
* Real-time analytics ETL
* Stream big data to data lake (S3)
* Search indexing (OpenSearch)
* Redshift batch loading
* Metric pipelines
* Mobile/IoT event ingestion

---

# **🔥 Developer Examples (Short)**

### **Send events directly to Firehose — Node.js**

```js
import { FirehoseClient, PutRecordCommand } from "@aws-sdk/client-firehose";

const client = new FirehoseClient({ region: "us-east-1" });

await client.send(new PutRecordCommand({
  DeliveryStreamName: "my-firehose-stream",
  Record: {
    Data: Buffer.from(JSON.stringify({ user: 1, action: "click" }) + "\n")
  }
}));
```

---

### **Firehose Lambda Transformation (Node.js)**

```js
export const handler = async (event) => {
  return {
    records: event.records.map(r => ({
      recordId: r.recordId,
      result: "Ok", 
      data: Buffer.from(
        JSON.stringify({ ...JSON.parse(Buffer.from(r.data, 'base64')), ts: Date.now() })
      ).toString('base64')
    }))
  };
};
```

---

# **🔥 Additional SDE-3 Questions**

### **How did you tune Firehose buffer interval and size?**

E.g., Buffer Size = 5MB, Interval = 60 seconds for near-real-time.
For cost optimization, increase buffer = fewer writes, lower Redshift COPY usage.

---

### **Did you enable dynamic partitioning in Firehose? Why?**

Use when writing large, analytics-driven S3 datasets using folders like:
`/year=2025/month=02/day=15/region=ap-south-1/`.

---

### **What was your backup S3 strategy?**

Store all failures (backupMode = All) + lifecycle rules + Glue crawler for error inspection.

---

### **How did you handle Redshift COPY failures in Firehose?**

By monitoring delivery failures, inspecting rejected rows from S3, and validating schema via Glue/Athena.

---

### **How did you integrate Firehose with Kinesis Data Streams?**

Firehose sourced from a Kinesis Stream when custom real-time transformation was required before batching and delivery.

---


