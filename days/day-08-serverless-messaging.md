# Day 8 — Decoupling & Serverless: SQS, SNS, EventBridge, Kinesis, Lambda, API Gateway, Step Functions

**Domain weight:** mainly *Design Resilient Architectures* (26% — "decouple", "loosely coupled", "survive spikes") and *Design High-Performing Architectures* (24% — streaming, scaling, serverless). Security (30%) shows up through resource policies, authorizers and encryption; Cost (20%) through "serverless instead of idle EC2".
**Time plan:** Learn 90 min · Lab 30 min · Drill 45 min · Review 30 min.

By the end of today you should be able to:

- Pick between SQS, SNS, EventBridge, Kinesis Data Streams, Amazon Data Firehose and Amazon MQ from a single scenario sentence.
- Explain visibility timeout, dead-letter queues, long polling and FIFO message groups — and fix the classic "messages processed twice" and "poison message" problems.
- Describe the three ways Lambda is invoked (sync, async, event source mapping) and what happens on failure in each.
- Size Lambda concurrency and fix cold starts, VPC internet access and database connection storms.
- Choose an API Gateway type, endpoint type and authorizer, and know when API Gateway talks directly to AWS services.
- Choose Step Functions Standard vs Express and recognise retry/catch, Map, and wait-for-callback patterns.

---

## 1. Why decouple? The mental model

A tightly coupled system calls the next tier synchronously. If the next tier is slow or down, the caller fails too, and a traffic spike hits every tier at once.

```
Tightly coupled                         Decoupled
┌──────┐  HTTP  ┌──────────┐            ┌──────┐   ┌───────────┐   ┌──────────┐
│ Web  │──────► │ Workers  │            │ Web  │──►│ SQS queue │◄──│ Workers  │ (ASG scales on
└──────┘        └──────────┘            └──────┘   └───────────┘   └──────────┘  queue depth)
spike → workers overwhelm → web errors   spike → queue absorbs it → workers catch up
```

Four integration styles, four services:

| Style | Shape | Service | One-line test |
| --- | --- | --- | --- |
| **Queue** (point-to-point) | 1 producer → buffer → 1 logical consumer | **SQS** | "Buffer work, process each item once, consumers pull" |
| **Pub/sub** (push) | 1 message → many subscribers | **SNS** | "Notify many systems at once" |
| **Event bus** (routing) | Events matched by content rules → targets | **EventBridge** | "React to AWS/SaaS events, route by content, schedule" |
| **Stream** (log) | Ordered, replayable records, many readers | **Kinesis Data Streams** (or MSK) | "Real-time, ordered, multiple consumers, replay" |

Plus **orchestration** (Step Functions — a central coordinator runs the steps) vs **choreography** (services react to each other's events via SNS/EventBridge). Exam cue: "coordinate multiple Lambda functions with retries, branching and human approval" → Step Functions.

---

## 2. Amazon SQS

Fully managed queue. Producers `SendMessage`; consumers **poll** with `ReceiveMessage`, process, then `DeleteMessage`. Nothing is pushed.

### The message lifecycle (the heart of every SQS question)

```
 SendMessage           ReceiveMessage                    DeleteMessage
     │                      │                                  │
     ▼                      ▼                                  ▼
 [ visible ] ──────► [ in flight: invisible for the ] ───► [ gone ]
     ▲                [ VISIBILITY TIMEOUT           ]
     │                         │ timeout expires, not deleted
     └─────────────────────────┘  → visible again → redelivered (ReceiveCount + 1)
                                  after maxReceiveCount → moved to DEAD-LETTER QUEUE
```

- **Visibility timeout** (default 30 s, max 12 h): how long a received message stays hidden. Set it **longer than your processing time**. Too short → another consumer gets the same message → duplicates. A slow consumer can call `ChangeMessageVisibility` to extend it.
- **Dead-letter queue (DLQ):** a redrive policy with `maxReceiveCount` moves a message that keeps failing ("poison pill") aside so it stops blocking/repeating. Inspect it, fix the bug, then use **DLQ redrive** to send messages back to the source queue. A FIFO queue's DLQ must also be FIFO.
- **Retention:** default 4 days, configurable from 1 minute to 14 days. Messages older than that are deleted, even in the DLQ — set the DLQ's retention longer than the source's.
- **Long polling:** `WaitTimeSeconds` 1–20 s. The call waits for messages instead of returning empty. Fewer empty responses → lower cost and latency. Exam answer to "reduce the number of empty ReceiveMessage calls / reduce cost".
- **Delay queues / message timers:** postpone delivery up to 15 minutes.
- **Message size:** limited (historically 256 KB; AWS has raised it — **check current docs**). For bigger payloads use the **Extended Client Library** pattern: payload in S3, pointer in the message.
- **Encryption:** SSE-SQS (on by default for new queues) or SSE-KMS for customer-managed keys; in transit via HTTPS. Access control via IAM and a **queue policy** (resource-based — e.g. let an SNS topic or S3 bucket send to the queue).

### Standard vs FIFO

| | Standard | FIFO (`name.fifo`) |
| --- | --- | --- |
| Throughput | Nearly unlimited | Limited per queue/API action; higher with batching and **high-throughput mode** — check current docs for numbers |
| Delivery | **At-least-once** (occasional duplicates) | **Exactly-once processing** (5-minute deduplication window via `MessageDeduplicationId` or content-based dedup) |
| Ordering | Best-effort | Strict **within a message group** (`MessageGroupId`) |
| Use for | Most decoupling, max scale | Order matters (financial transactions, commands in sequence), no duplicates allowed |

How FIFO scales: ordering is per **message group**. Different groups are processed in parallel; within a group one message at a time. Use e.g. `customerId` as the group ID → per-customer ordering with parallelism across customers.

> Trap: "Converting a Standard queue to FIFO" — you can't. Create a new FIFO queue and point producers at it.
> Trap: Standard queue + "must never process duplicates" → make the consumer **idempotent** (e.g. DynamoDB conditional write on a message ID) or move to FIFO.

### Scaling consumers on queue depth

The canonical exam architecture: EC2 Auto Scaling group consumers, scaled by a **target tracking policy on backlog per instance** (custom metric = `ApproximateNumberOfMessagesVisible` ÷ running instances). Simpler answers that also appear: step scaling on a CloudWatch alarm on `ApproximateNumberOfMessagesVisible` or `ApproximateAgeOfOldestMessage`.

```bash
aws sqs create-queue --queue-name orders-dlq
aws sqs create-queue --queue-name orders --attributes '{
  "VisibilityTimeout": "180",
  "ReceiveMessageWaitTimeSeconds": "20",
  "RedrivePolicy": "{\"deadLetterTargetArn\":\"arn:aws:sqs:ap-south-1:111122223333:orders-dlq\",\"maxReceiveCount\":\"5\"}"
}'
```

### Amazon MQ (the "don't rewrite the app" queue)

Managed **Apache ActiveMQ** and **RabbitMQ**. Speaks industry protocols — JMS, AMQP, MQTT, STOMP, OpenWire. Exam cue: "migrate an on-premises application that uses JMS / AMQP / RabbitMQ **without code changes**" → Amazon MQ. Otherwise SQS/SNS (more scalable, fully serverless). MQ runs on brokers; for HA use an active/standby broker across two AZs.

---

## 3. Amazon SNS

Pub/sub. A publisher sends to a **topic**; SNS **pushes** a copy to every subscription. No persistence: if a subscriber is down, SNS retries according to its delivery policy, then the message is lost unless the subscription has a **DLQ** (an SQS queue attached to the subscription).

Subscriber types: **SQS**, **Lambda**, **HTTP/S**, **email**, **SMS**, **mobile push**, **Amazon Data Firehose**.

### Fan-out: SNS → many SQS queues

```
                      ┌──► SQS: fulfilment  ──► workers
 Order service ─► SNS ├──► SQS: analytics   ──► Lambda
   (publish once)     └──► SQS: email-notify ──► workers   (filter: only "priority=high")
```

Why not publish to three queues directly? The producer would need to know every consumer. With fan-out, adding a consumer is just a new subscription. Each queue buffers independently, so one slow consumer doesn't affect the others.

Requirements: the **SQS queue policy** must allow `sns.amazonaws.com` to `sqs:SendMessage` with `aws:SourceArn` = the topic. If the queue uses SSE-KMS with a customer managed key, the **KMS key policy** must also let SNS use the key — a favourite "messages never arrive" trap.

**Message filtering:** a **subscription filter policy** (on message attributes or the message body) lets each subscriber receive only the messages it cares about — no need for separate topics or consumer-side filtering.

```json
{ "eventType": ["order_placed"], "priority": ["high"] }
```

**SNS FIFO topics** preserve ordering and deduplicate, and deliver to SQS queues (fan-out with ordering). **Message archiving and replay** is available for FIFO topics.

**S3 event notifications** can target SNS, SQS, Lambda — or send everything to **EventBridge** for richer routing. Classic pattern: S3 upload → SNS → multiple SQS queues (one S3 event, many processors). S3 can't send the same event type/prefix to two destinations directly, which is exactly why fan-out exists.

---

## 4. Amazon EventBridge

A serverless **event bus**. Events are JSON; **rules** match them by **event pattern** and send them to **targets** (Lambda, SQS, SNS, Step Functions, Kinesis, API destinations, another bus, ECS tasks and many more).

```
 Sources                         Event bus                 Rules (pattern match)       Targets
 AWS services (EC2 state,  ─┐                         ┌─ source=aws.ec2, state=stopped ─► Lambda
 GuardDuty findings, S3…)   ├──►  default / custom  ──┤
 Your apps (PutEvents)  ────┤        bus              ├─ detail-type=OrderPlaced ──────► Step Functions
 SaaS partners (Zendesk,  ──┘                         └─ (cross-account) ─────────────► bus in acct B
 Datadog, Shopify…)
```

```json
{
  "source": ["aws.ec2"],
  "detail-type": ["EC2 Instance State-change Notification"],
  "detail": { "state": ["stopped", "terminated"] }
}
```

Features to recognise:

| Feature | What it gives you | Exam cue |
| --- | --- | --- |
| **Default bus** | Receives AWS service events automatically | "React when an EC2 instance stops / a GuardDuty finding appears" |
| **Partner event sources** | Native SaaS integrations | "Ingest events from a SaaS provider with no polling code" |
| **Content-based rules** | Filter on any field, prefix, numeric range, etc. | "Route events to different targets based on content" |
| **EventBridge Scheduler** | One-time or recurring (cron/rate) invocations at scale, with time zones | "Run a Lambda every night at 02:00" — replaces cron on EC2 |
| **Archive and replay** | Store events, replay later | "Reprocess past events after fixing a bug" |
| **Schema registry** | Discover event schemas, generate code bindings | Rare |
| **EventBridge Pipes** | Point-to-point source → filter → enrich → target (e.g. SQS/DynamoDB Streams/Kinesis → Step Functions) without glue Lambda | "Connect a stream to a target with filtering and no custom code" |
| **Cross-account / cross-Region buses** | Send events to a bus in another account or Region | Central security/ops account |
| **API destinations** | Call any HTTP API as a target, with auth and rate limiting | "Send events to a third-party webhook" |

**SNS vs EventBridge:** both fan out. SNS → very high throughput, simple attribute/body filtering, push to SMS/email/mobile. EventBridge → AWS service and SaaS events, richer pattern matching, schedules, archive/replay, many more target types. If the source is an AWS service event or a SaaS partner, think EventBridge.

> Note: EventBridge is the evolution of "CloudWatch Events". Old questions may say CloudWatch Events — same thing.

---

## 5. Kinesis family and streaming

### Kinesis Data Streams (KDS)

A durable, ordered, replayable stream. Capacity is in **shards**.

```
Producers (SDK, KPL, agent)
   │   PutRecord(partitionKey="device-42", data)
   ▼   hash(partitionKey) → shard
┌──────── Stream ────────────────────────────────┐
│ Shard 1: rec rec rec rec ──►                    │   retention: 24 h default, extendable up to 365 days
│ Shard 2: rec rec rec ──►                        │   records are NOT deleted when read
│ Shard 3: rec rec rec rec rec ──►                │
└─────────────────────────────────────────────────┘
   │            │                 │
 Lambda     KCL app on EC2     Managed Service for Apache Flink / Firehose
 (each consumer reads independently, at its own position)
```

- **Per-shard capacity** (provisioned mode): about 1 MB/s or 1,000 records/s **in**, 2 MB/s **out** shared across standard consumers. **Enhanced fan-out** gives each registered consumer its own 2 MB/s per shard with push delivery — use it with many consumers or for low latency. (Check current docs for exact quotas.)
- **Ordering** is guaranteed **within a shard**; records with the same partition key go to the same shard. A skewed key → **hot shard** → `ProvisionedThroughputExceededException`. Fix: a higher-cardinality partition key, or split shards.
- **Capacity modes:** **on-demand** (auto-scales, pay per throughput — unpredictable traffic) vs **provisioned** (you set shard count — predictable, cheaper at steady scale).
- **Replay:** consumers can re-read from any point within retention. That's the key difference from SQS.
- Security: KMS encryption at rest, IAM, VPC interface endpoint.

### Amazon Data Firehose (formerly Kinesis Data Firehose)

Fully managed **delivery** service: takes records and lands them in a destination — **S3, Redshift (via S3 COPY), OpenSearch Service, Splunk, Snowflake, Apache Iceberg tables, HTTP endpoints** and partner tools.

- **Near real-time**, not real-time: it **buffers** by size or time before writing (buffer interval can be set very low, even zero, but it's still a delivery service, not a stream you query).
- No shards, no consumers to manage, auto-scales. No replay/retention for you to read from.
- Optional **Lambda transformation**, **record format conversion** to Parquet/ORC (with a Glue Data Catalog schema), **dynamic partitioning** of S3 prefixes, compression.
- Sources: direct PUT, a Kinesis data stream, MSK, CloudWatch Logs subscriptions, AWS IoT, SNS.

### Others in the family

| Service | Use |
| --- | --- |
| **Amazon Managed Service for Apache Flink** (formerly Kinesis Data Analytics) | Real-time SQL/Java/Python analytics on streams — windows, aggregations, anomaly detection |
| **Kinesis Video Streams** | Ingest/store video from cameras for playback or ML (Rekognition) |
| **Amazon MSK** | Managed Apache Kafka (provisioned or Serverless). Cue: "already uses Kafka", "Kafka APIs", very large messages, open-source compatibility |

### Kinesis Data Streams vs Firehose vs SQS

| | Kinesis Data Streams | Amazon Data Firehose | SQS |
| --- | --- | --- | --- |
| Purpose | Real-time stream processing | Load streaming data into storage/analytics | Decouple work items |
| Latency | ~sub-second (real time) | Seconds to minutes (buffered) | Milliseconds; consumer-paced |
| Consumers | Many, each independent | Destination only | Competing consumers; message gone after delete |
| Replay | Yes, within retention | No | No (deleted once processed) |
| Ordering | Per shard (partition key) | n/a | FIFO queues only |
| Ops | Shards (or on-demand) | None | None |
| Custom code | Consumers you write | Optional Lambda transform | Consumers you write |

Typical pipelines:

```
Clickstream ─► Kinesis Data Streams ─┬─► Lambda (real-time fraud flag)
                                     ├─► Managed Flink (5-min rolling metrics)
                                     └─► Firehose ─► S3 (Parquet) ─► Athena / Redshift

IoT / logs ─► Firehose ─► S3   (simplest "ingest and store for analytics" answer)
```

---

## 6. AWS Lambda

Run code without servers. You pay per request and per GB-second of duration. It scales by running more copies (**concurrency**) in parallel.

### Key limits (verify numbers in current docs — they change)

| Setting | Value to remember |
| --- | --- |
| Max timeout | **15 minutes** — anything longer → ECS/Fargate, AWS Batch, or split with Step Functions |
| Memory | 128 MB – 10,240 MB; **CPU scales with memory** (more memory = faster CPU-bound code) |
| Ephemeral `/tmp` | 512 MB default, configurable up to 10 GB |
| Package | Zip (with size limits) or **container image** up to 10 GB; shared code via **layers** |
| Concurrency | Account-level regional quota (default 1,000; can be raised) |

### Three invocation models — and what happens on failure

| Model | Who calls | Examples | On error |
| --- | --- | --- | --- |
| **Synchronous** | Caller waits for the response | API Gateway, ALB, function URLs, SDK `Invoke`, Cognito triggers | Error goes back to the caller; **caller** must retry |
| **Asynchronous** | Lambda queues the event internally, returns 202 | S3, SNS, EventBridge, CloudWatch Logs, SES | Lambda retries (up to 2 more times by default) then sends to an **on-failure destination** (SQS, SNS, Lambda, EventBridge, S3) or a DLQ. Prefer **destinations** — they also support on-success and carry more context |
| **Event source mapping** (polling) | Lambda polls the source for you | **SQS, Kinesis, DynamoDB Streams, MSK**, Amazon MQ, DocumentDB | SQS: message returns after visibility timeout → eventually DLQ on the queue. Streams: retries the batch until success/expiry, **blocking the shard** — use `BisectBatchOnFunctionError`, `MaximumRetryAttempts`, `MaximumRecordAgeInSeconds` and an on-failure destination |

SQS + Lambda specifics:

- Set the queue's **visibility timeout ≥ 6 × the function timeout** (AWS guidance) so in-flight batches don't reappear while Lambda is still retrying.
- Return **partial batch responses** (`ReportBatchItemFailures`) so one bad message doesn't make the whole batch retry.
- Use **maximum concurrency** on the SQS event source to cap how hard Lambda hits a downstream system.

### Concurrency, cold starts and scaling

```
concurrency ≈ requests per second × average duration (s)
e.g. 200 req/s × 0.5 s = 100 concurrent executions
```

| Control | Effect | Use when |
| --- | --- | --- |
| **Reserved concurrency** | Guarantees *and caps* a function's concurrency; 0 = throttle completely | Protect a downstream DB; stop one function starving others |
| **Provisioned concurrency** | Keeps N execution environments **pre-initialised** → no cold start (costs money while idle) | Latency-sensitive APIs, predictable peaks (schedule with Application Auto Scaling) |
| **SnapStart** | Snapshots the initialised environment and resumes from it → much faster cold starts (Java, Python, .NET — check current runtimes) | Java functions with slow init, without paying for provisioned concurrency |

Throttling: once the account or reserved limit is reached, sync calls get **429 TooManyRequestsException**; async events are retried for up to 6 hours.

### Lambda networking

- By default Lambda runs in an AWS-managed VPC with **internet access** but **no access to your private resources**.
- Attach it to **your VPC subnets** (with a security group) to reach RDS, ElastiCache, internal ALBs. Lambda uses shared Hyperplane ENIs — no per-invocation ENI cold-start penalty any more.
- Once in your VPC it **loses internet access**. Fix: put it in **private subnets with a route to a NAT gateway** (a public subnet + IGW does **not** work — Lambda ENIs have no public IP). To reach S3/DynamoDB privately use **gateway endpoints**; other services, **interface endpoints**.

> Trap: "Lambda in a VPC times out calling a third-party API" → NAT gateway in a public subnet, Lambda in private subnets routing to it.

### Lambda + relational databases

Hundreds of concurrent functions each opening a DB connection exhausts RDS. Fix: **Amazon RDS Proxy** — pools and shares connections, improves failover time, and can use IAM auth + Secrets Manager. Also open the connection **outside the handler** so warm invocations reuse it.

### Other Lambda facts the exam uses

- **Execution role** grants Lambda's permissions; a **resource-based policy** lets other services/accounts invoke it (e.g. S3, SNS, API Gateway).
- **Environment variables** are encrypted at rest with KMS; secrets belong in **Secrets Manager / Parameter Store** (use the Parameters and Secrets extension to cache them).
- **Versions** are immutable; **aliases** (`prod`, `beta`) point to versions and support **weighted traffic shifting** — with CodeDeploy for canary/linear deployments.
- **Function URLs:** a dedicated HTTPS endpoint for one function (auth = IAM or none). Simpler than API Gateway, but no usage plans, caching or request validation.
- **Lambda@Edge / CloudFront Functions:** code at edge locations — Day 9.
- **Graviton (arm64):** typically better price-performance — a cost-optimisation answer.

---

## 7. Amazon API Gateway

Managed front door for APIs: auth, throttling, validation, transformation, caching, monitoring.

### API types

| | **REST API** | **HTTP API** | **WebSocket API** |
| --- | --- | --- | --- |
| Positioning | Full feature set | Cheaper, lower latency, simpler | Persistent two-way connections |
| Auth | IAM, Cognito user pools, Lambda authorizers, resource policies | IAM, **JWT authorizers** (Cognito/any OIDC), Lambda authorizers | IAM, Lambda authorizers |
| Caching | **Yes** | No | No |
| Usage plans + API keys | **Yes** | No | No |
| Request validation, mapping templates | Yes | Limited (parameter mapping) | Route selection |
| AWS WAF | **Yes** | Not directly (use CloudFront in front) | No |
| Private endpoints | **Yes** | No | No |
| Use for | Public/partner APIs needing keys, quotas, caching, WAF, private APIs | Simple Lambda/HTTP proxy APIs at lowest cost | Chat, live dashboards, games |

### Endpoint types (REST)

| Type | Where | Use |
| --- | --- | --- |
| **Edge-optimized** | Routed through CloudFront PoPs | Geographically distributed clients |
| **Regional** | In the Region | Clients in the same Region, or you want your own CloudFront distribution in front |
| **Private** | Only reachable via an **interface VPC endpoint** (`execute-api`), controlled with a resource policy | Internal APIs that must not be on the internet |

### Integrations

- **Lambda proxy** — the most common serverless API.
- **HTTP** — proxy to any public HTTP endpoint.
- **AWS service integration** — call SQS `SendMessage`, DynamoDB `PutItem`, Kinesis `PutRecord`, Step Functions `StartExecution` **directly, with no Lambda in between**. Cue: "reduce latency/cost/operational overhead — remove the Lambda that just forwards to SQS".
- **VPC link** — reach private resources: REST API → **NLB** (and newer support for ALB — check current docs); HTTP API → ALB, NLB or Cloud Map.
- **Mock** — return static responses.

### Protection and performance

- **Throttling:** account-level per-Region steady-state and burst limits (token bucket), plus per-stage/per-method limits and per-client limits via **usage plans** with **API keys**. Over the limit → **429**. API keys identify/meter clients; they are **not** an authentication mechanism.
- **Caching** (REST): per-stage cache with TTL (default 300 s), reduces backend calls.
- **Integration timeout:** 29 seconds by default (can be increased for Regional/private REST APIs — check current docs). Long work → return immediately and process asynchronously (SQS/Step Functions), or use WebSockets/polling for the result.
- **Payload limit** ~10 MB → large uploads go straight to S3 with a **pre-signed URL**.
- **Stages** (`dev`, `prod`) with **stage variables** and **canary releases**.
- Logging/metrics in CloudWatch; tracing with **X-Ray**.

### Authorizers — which one?

| Need | Authorizer |
| --- | --- |
| End users sign in to your app (sign-up, social login) | **Cognito user pool** authorizer (REST) / **JWT authorizer** (HTTP) |
| Other AWS principals / services call the API with SigV4 | **IAM** authorization |
| Custom token format, third-party IdP, header/query logic | **Lambda authorizer** (token or request-based; results cacheable) |
| Restrict by source VPC endpoint, account or IP | **Resource policy** (REST) |

### Serverless reference architecture

```
Client ─► (CloudFront + WAF) ─► API Gateway ──► Lambda ──► DynamoDB
                │ Cognito JWT                    │
                │                                └─► SQS ─► Lambda (async work)
                └─► direct integration ─► Step Functions (long workflow)
```

---

## 8. AWS Step Functions

Serverless **orchestration** with state machines defined in Amazon States Language (JSON/YAML) or the visual Workflow Studio. Each step can call Lambda or — via **SDK integrations** — over 200 AWS services directly (DynamoDB, SQS, SNS, ECS, Glue, SageMaker, Bedrock…).

### Workflow types

| | **Standard** | **Express** |
| --- | --- | --- |
| Max duration | Up to **1 year** | Up to **5 minutes** |
| Execution semantics | **Exactly-once** | At-least-once (async) / at-most-once (sync) |
| Execution history | Full visual history kept | CloudWatch Logs only |
| Pricing model | Per **state transition** | Per execution + duration/memory |
| Use | Long-running, auditable business processes, human approval | High-volume, short event processing (IoT ingest, streaming transforms, API backends) |

### States and patterns

| State / feature | Purpose |
| --- | --- |
| **Task** | Do work (Lambda, SDK call, activity) |
| **Choice** | Branch on input |
| **Parallel** | Run branches concurrently, join |
| **Map** | Loop over items; **Distributed Map** processes millions of S3 objects/items with high concurrency |
| **Wait** | Pause for a duration or until a timestamp (free waiting in Standard — no compute) |
| **Pass / Succeed / Fail** | Shape data, end |
| **Retry / Catch** | Declarative retries with backoff; route errors to compensation steps (saga pattern) |
| **`.waitForTaskToken`** (callback) | Pause until an external system/human calls `SendTaskSuccess` — "manager approval" workflows |
| **`.sync`** | Start a job (ECS task, Glue job, Batch) and wait for it to finish |

```json
{
  "StartAt": "ChargeCard",
  "States": {
    "ChargeCard": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": { "FunctionName": "charge-card", "Payload.$": "$" },
      "Retry": [{ "ErrorEquals": ["States.TaskFailed"], "IntervalSeconds": 2, "MaxAttempts": 3, "BackoffRate": 2 }],
      "Catch": [{ "ErrorEquals": ["States.ALL"], "Next": "RefundAndNotify" }],
      "Next": "WaitForApproval"
    },
    "WaitForApproval": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sqs:sendMessage.waitForTaskToken",
      "Parameters": { "QueueUrl": "https://sqs.ap-south-1.amazonaws.com/111122223333/approvals",
                      "MessageBody": { "token.$": "$$.Task.Token", "order.$": "$" } },
      "Next": "Done"
    },
    "RefundAndNotify": { "Type": "Fail" },
    "Done": { "Type": "Succeed" }
  }
}
```

Why Step Functions instead of Lambda calling Lambda? No 15-minute ceiling for the whole workflow, retries/error handling are configuration not code, waiting costs nothing, and you get a visual audit trail.

Related names you may see: **AWS SWF** (legacy predecessor — rarely the right answer), **Amazon MWAA** (managed Airflow — data pipelines already in Airflow), **AWS AppSync** (managed GraphQL/real-time APIs — cue "GraphQL").

---

## 9. Exam traps roundup

1. **Duplicates in SQS Standard** → visibility timeout too short, or no idempotency. "Exactly once + ordered" → FIFO.
2. **Poison message retried forever** → DLQ with `maxReceiveCount`.
3. **Lots of empty receives / high SQS cost** → long polling (20 s).
4. **SNS messages lost when subscriber was down** → SNS → SQS fan-out (queues persist), or a subscription DLQ.
5. **SNS → encrypted SQS not delivering** → KMS key policy must allow SNS; queue policy must allow the topic.
6. **"Real-time" + "multiple consumers" + "replay"** → Kinesis Data Streams, not SQS.
7. **"Load streaming data into S3/Redshift/OpenSearch with no code/ops"** → Firehose. It's near real-time — if the question says "real-time processing in milliseconds", it's KDS.
8. **Hot shard** → better partition key / more shards.
9. **Job runs > 15 minutes** → not Lambda (Fargate, Batch, or Step Functions to split).
10. **API call > 29 s** → go async (SQS / Step Functions) — don't just "increase the Lambda timeout".
11. **Lambda in VPC can't reach internet** → private subnet + NAT gateway (not IGW).
12. **Lambda exhausting DB connections** → RDS Proxy (and/or reserved concurrency).
13. **Cold starts on a latency-critical API** → provisioned concurrency (or SnapStart for Java).
14. **API keys as auth** → wrong; keys are for usage plans. Use Cognito/IAM/Lambda authorizers.
15. **JMS/AMQP app migration with no code change** → Amazon MQ, not SQS.
16. **Cron on a single EC2 instance (SPOF)** → EventBridge Scheduler → Lambda/ECS task.
17. **Kinesis stream consumer falls behind with Lambda** → increase shards / parallelization factor; check batch size and errors blocking the shard.

---

## 10. Exam decision table

| If the question says… | Pick |
| --- | --- |
| Decouple tiers, absorb spikes, workers process at their own pace | **SQS Standard** (+ ASG scaling on queue depth) |
| Order must be preserved and no duplicates | **SQS FIFO** (message group ID for parallelism) |
| Message fails repeatedly / poison message | **DLQ** with redrive policy; redrive back after the fix |
| Reduce empty receives / SQS cost | **Long polling** |
| Payload larger than SQS limit | Store in **S3**, send pointer (Extended Client Library) |
| One event must reach several independent consumers | **SNS → multiple SQS queues** (fan-out) |
| Each subscriber only wants some messages | **SNS subscription filter policy** (or EventBridge rules) |
| Push notifications to email/SMS/mobile | **SNS** |
| React to AWS service events (EC2 state change, GuardDuty finding, Config change) | **EventBridge rule** |
| Ingest SaaS partner events without polling | **EventBridge partner event source** |
| Run something on a schedule, serverless | **EventBridge Scheduler** |
| Reprocess past events after a bug fix | **EventBridge archive & replay** (or KDS replay for streams) |
| Real-time stream, ordered per key, multiple consumers, replay | **Kinesis Data Streams** |
| Load streaming data into S3/Redshift/OpenSearch/Splunk, minimal ops | **Amazon Data Firehose** |
| Convert streaming JSON to Parquet on the way into S3 | **Firehose record format conversion** |
| Real-time analytics/windowed aggregations on a stream | **Managed Service for Apache Flink** |
| Existing Kafka workloads | **Amazon MSK** |
| Existing JMS/AMQP/MQTT/RabbitMQ app, no code changes | **Amazon MQ** |
| Short event-driven compute, unpredictable traffic, pay per use | **Lambda** |
| Task longer than 15 minutes | **Fargate / AWS Batch** (or Step Functions orchestration) |
| Lambda cold starts hurt latency | **Provisioned concurrency** (Java: **SnapStart**) |
| Stop a function overwhelming a downstream system | **Reserved concurrency** / SQS event source max concurrency |
| Many Lambdas → RDS "too many connections" | **RDS Proxy** |
| Lambda in VPC needs internet | **NAT gateway**, Lambda in private subnets |
| Serverless REST API with caching, API keys, usage plans, WAF | **API Gateway REST API** |
| Simple, low-cost Lambda/HTTP proxy API with JWT auth | **API Gateway HTTP API** |
| Real-time two-way (chat, live updates) | **API Gateway WebSocket API** (or AppSync for GraphQL subscriptions) |
| Internal-only API reachable from a VPC | **Private REST API** + interface VPC endpoint |
| API that just writes to SQS/DynamoDB — remove the Lambda | **API Gateway AWS service integration** |
| Users sign up/sign in, API validates tokens | **Cognito user pool authorizer** |
| Clients upload large files through the API | **S3 pre-signed URL** instead |
| Coordinate multiple steps with retries, branching, human approval | **Step Functions Standard** (`.waitForTaskToken`) |
| High-volume, short (< 5 min) workflows | **Step Functions Express** |
| Process millions of S3 objects in parallel | **Step Functions Distributed Map** |
| GraphQL API | **AWS AppSync** |

---

## 11. Lab (30 min, free tier)

Goal: build an SNS → SQS fan-out with a filter policy, consume one queue with Lambda through an event source mapping, and watch a poison message land in the DLQ.

**Cost note:** SNS, SQS and Lambda all have free-tier/always-free allowances that cover this lab many times over — but **check current free-tier terms** for your account. Nothing here runs when idle except what you forget to delete.

```
            ┌──► SQS saa-all      (every message)            — you read it by hand
SNS saa-orders
            └──► SQS saa-priority (filter: priority=high) ──► Lambda saa-worker
                        │ maxReceiveCount = 2
                        └──► SQS saa-priority-dlq
```

### Console steps

1. **SQS** → *Create queue* → Standard, name `saa-priority-dlq`, retention **14 days**. Create.
2. *Create queue* → Standard, `saa-priority`. Visibility timeout **30 s**. *Receive message wait time* **20 s** (long polling). *Dead-letter queue* → enable, choose `saa-priority-dlq`, **maximum receives 2**. Create.
3. *Create queue* → Standard, `saa-all`, defaults. Create.
4. **SNS** → *Create topic* → Standard, `saa-orders`.
5. *Create subscription* → protocol **Amazon SQS** → `saa-all`. Create. Then another → `saa-priority`, enable **raw message delivery**, and under *Subscription filter policy* (scope: message attributes):
   ```json
   { "priority": ["high"] }
   ```
   When you subscribe from the SNS console to a queue in the same account, the console adds the queue policy that lets the topic send. Open `saa-priority` → *Access policy* and find the `sns.amazonaws.com` / `aws:SourceArn` statement.
6. **Lambda** → *Create function* → `saa-worker`, Python 3.x (latest offered), arm64. Timeout **5 s** (queue visibility 30 s = 6 × function timeout). Code:
   ```python
   # Console default handler: lambda_function.lambda_handler
   def lambda_handler(event, context):
       failures = []
       for r in event["Records"]:
           body = r["body"]
           print("got:", body)
           if "poison" in body:
               failures.append({"itemIdentifier": r["messageId"]})
       # Partial batch response: only the failed messages go back to the queue
       return {"batchItemFailures": failures}
   ```
   Deploy. Under *Configuration → Permissions*, attach `AWSLambdaSQSQueueExecutionRole` to the execution role (lets Lambda poll and delete).
7. *Add trigger* → **SQS** → `saa-priority`, batch size 10, tick **Report batch item failures**. Add.
8. Publish test messages from **CloudShell**:
   ```bash
   TOPIC=$(aws sns list-topics --query "Topics[?ends_with(TopicArn,':saa-orders')].TopicArn" --output text)
   aws sns publish --topic-arn $TOPIC --message '{"order":"O1"}' \
     --message-attributes '{"priority":{"DataType":"String","StringValue":"low"}}'
   aws sns publish --topic-arn $TOPIC --message '{"order":"O2"}' \
     --message-attributes '{"priority":{"DataType":"String","StringValue":"high"}}'
   aws sns publish --topic-arn $TOPIC --message '{"order":"poison"}' \
     --message-attributes '{"priority":{"DataType":"String","StringValue":"high"}}'
   ```
9. Observe:
   - `saa-all` → *Send and receive messages* → *Poll* → **3** messages (wrapped in the SNS JSON envelope, since raw delivery is off there).
   - Lambda → *Monitor → View CloudWatch logs* → `O2` processed once; `poison` appears twice.
   - After ~1 minute, `saa-priority-dlq` has **1** message (`poison`) — it failed `maxReceiveCount` times. `O1` never reached `saa-priority`: the filter policy dropped it.
10. Optional: on `saa-priority-dlq` click **Start DLQ redrive** → back to source queue. It will fail again and return — the point is the redrive workflow exists for after you fix the bug.
11. Optional (5 min): **EventBridge → Scheduler → Create schedule**, rate `5 minutes`, target *SNS Publish* to `saa-orders` with a fixed message. Watch `saa-all` fill up, then **delete the schedule**.
12. **Tear down:** delete the Lambda trigger and function (and its CloudWatch log group `/aws/lambda/saa-worker`), the SNS topic (subscriptions go with it), the three queues, any EventBridge schedule, and the execution role the console created.

### Terraform variant

```hcl
provider "aws" { region = "ap-south-1" }

resource "aws_sqs_queue" "dlq" {
  name                      = "saa-priority-dlq"
  message_retention_seconds = 1209600 # 14 days
}

resource "aws_sqs_queue" "priority" {
  name                       = "saa-priority"
  visibility_timeout_seconds = 30
  receive_wait_time_seconds  = 20 # long polling
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.dlq.arn
    maxReceiveCount     = 2
  })
}

resource "aws_sqs_queue" "all" { name = "saa-all" }

resource "aws_sns_topic" "orders" { name = "saa-orders" }

# Queue policy: allow only this topic to send
data "aws_iam_policy_document" "from_sns" {
  for_each = { priority = aws_sqs_queue.priority.arn, all = aws_sqs_queue.all.arn }
  statement {
    actions   = ["sqs:SendMessage"]
    resources = [each.value]
    principals {
      type        = "Service"
      identifiers = ["sns.amazonaws.com"]
    }
    condition {
      test     = "ArnEquals"
      variable = "aws:SourceArn"
      values   = [aws_sns_topic.orders.arn]
    }
  }
}

resource "aws_sqs_queue_policy" "priority" {
  queue_url = aws_sqs_queue.priority.id
  policy    = data.aws_iam_policy_document.from_sns["priority"].json
}

resource "aws_sqs_queue_policy" "all" {
  queue_url = aws_sqs_queue.all.id
  policy    = data.aws_iam_policy_document.from_sns["all"].json
}

resource "aws_sns_topic_subscription" "all" {
  topic_arn = aws_sns_topic.orders.arn
  protocol  = "sqs"
  endpoint  = aws_sqs_queue.all.arn
}

resource "aws_sns_topic_subscription" "priority" {
  topic_arn            = aws_sns_topic.orders.arn
  protocol             = "sqs"
  endpoint             = aws_sqs_queue.priority.arn
  raw_message_delivery = true
  filter_policy        = jsonencode({ priority = ["high"] })
}

# ---------- Lambda consumer ----------
data "archive_file" "worker" {
  type        = "zip"
  output_path = "${path.module}/worker.zip"
  source {
    filename = "index.py"
    content  = <<-PY
      def handler(event, context):
          failures = []
          for r in event["Records"]:
              print("got:", r["body"])
              if "poison" in r["body"]:
                  failures.append({"itemIdentifier": r["messageId"]})
          return {"batchItemFailures": failures}
    PY
  }
}

resource "aws_iam_role" "worker" {
  name = "saa-worker-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{ Effect = "Allow", Action = "sts:AssumeRole",
                   Principal = { Service = "lambda.amazonaws.com" } }]
  })
}

resource "aws_iam_role_policy_attachment" "sqs_exec" {
  role       = aws_iam_role.worker.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaSQSQueueExecutionRole"
}

resource "aws_lambda_function" "worker" {
  function_name    = "saa-worker"
  role             = aws_iam_role.worker.arn
  runtime          = "python3.12" # use the latest runtime your provider supports
  architectures    = ["arm64"]
  handler          = "index.handler"
  filename         = data.archive_file.worker.output_path
  source_code_hash = data.archive_file.worker.output_base64sha256
  timeout          = 5 # queue visibility (30 s) >= 6 x timeout
}

resource "aws_lambda_event_source_mapping" "priority" {
  event_source_arn        = aws_sqs_queue.priority.arn
  function_name           = aws_lambda_function.worker.arn
  batch_size              = 10
  function_response_types = ["ReportBatchItemFailures"]
  scaling_config { maximum_concurrency = 5 } # cap pressure on downstream systems
}

output "topic_arn" { value = aws_sns_topic.orders.arn }
```

Run `terraform init && terraform apply`, repeat step 8 with `TOPIC=$(terraform output -raw topic_arn)`, observe as in step 9, then **`terraform destroy`** and delete the leftover log group `/aws/lambda/saa-worker` (Terraform didn't create it, so it won't remove it).

---

## 12. Practice questions

Answer all ten before opening the answers.

**Q1.** An e-commerce company's web tier sends orders synchronously to a fleet of EC2 processing instances. During flash sales the processing tier is overwhelmed and orders are lost. The company wants the system to absorb spikes and scale processing automatically. Which TWO actions should a solutions architect take? (Choose TWO.)
A. Have the web tier send orders to an Amazon SQS queue that the processing instances poll
B. Place the processing instances behind a Network Load Balancer
C. Scale the processing Auto Scaling group based on the queue's backlog per instance
D. Increase the processing instances to a larger instance type
E. Publish orders to an Amazon SNS topic with the processing instances subscribed over HTTP

**Q2.** A bank processes account transactions. Transactions for the same account must be processed in the exact order they were sent and never twice. Transactions for different accounts should be processed in parallel. What should the architect use?
A. SQS Standard queue with a Lambda consumer
B. SQS FIFO queue with the account ID as the message group ID
C. SNS Standard topic with an SQS subscription
D. Kinesis Data Firehose delivering to S3

**Q3.** When a user uploads an image to an S3 bucket, three independent systems must process it: thumbnail generation, metadata indexing and malware scanning. Each must be able to scale and fail independently, and new processors will be added later. Which design meets these requirements?
A. Configure three S3 event notifications for the same event type, each to a different Lambda function
B. Configure an S3 event notification to an SNS topic, with an SQS queue per processor subscribed to the topic
C. Have one Lambda function call the three processors sequentially
D. Use S3 Replication to copy objects to three buckets

**Q4.** A media company collects clickstream events from millions of users. A real-time fraud detector and a recommendation engine must both read every event independently with sub-second latency, and the company must be able to reprocess the last 7 days of events after deploying model changes. Which service should ingest the events?
A. Amazon SQS Standard queue
B. Amazon SNS topic
C. Amazon Kinesis Data Streams with retention set to 7 days
D. Amazon Data Firehose

**Q5.** Thousands of IoT sensors send JSON readings. The company wants the data stored in Amazon S3 in Apache Parquet format, partitioned by date, for querying with Amazon Athena. The solution must require the LEAST operational overhead. What should the architect recommend?
A. Kinesis Data Streams with a fleet of EC2 consumers that convert to Parquet
B. Amazon Data Firehose with record format conversion and dynamic partitioning, delivering to S3
C. Amazon SQS with a Lambda function that writes Parquet files
D. Amazon MSK with Kafka Connect on EC2

**Q6.** A Lambda function was attached to private subnets in a VPC so that it can query an Amazon RDS database. Since the change, the function's calls to an external payment provider's HTTPS API time out. What should the architect do?
A. Attach an internet gateway to the VPC and add a route from the private subnets to it
B. Assign an Elastic IP address to the Lambda function
C. Create a NAT gateway in a public subnet and route the private subnets' internet-bound traffic to it
D. Move the function out of the VPC and open the RDS database to the internet

**Q7.** A serverless application uses API Gateway and Lambda functions that query an Amazon RDS for MySQL database. During traffic spikes, the database rejects requests with "too many connections" errors. What is the MOST effective solution with minimal code changes?
A. Increase the Lambda function's memory
B. Put Amazon RDS Proxy between the functions and the database
C. Migrate the database to a larger instance class
D. Enable provisioned concurrency on the functions

**Q8.** Uploading a video to S3 must start a transcoding job that takes 30–60 minutes and uses a custom container. The company wants a serverless solution with the LEAST operational overhead. Which solution meets these requirements?
A. Trigger a Lambda function from the S3 event and set its timeout to 60 minutes
B. Use an EventBridge rule on the S3 Object Created event to run an Amazon ECS task on AWS Fargate
C. Launch an EC2 instance from a Lambda function for each upload
D. Use API Gateway to call the transcoder synchronously

**Q9.** An order workflow must charge a card, reserve inventory and arrange shipping, retrying failed steps with exponential backoff. Orders over a threshold require a manager's approval, which can take up to 3 days. Each order's history must be auditable. Which solution should the architect choose?
A. A chain of Lambda functions where each invokes the next
B. AWS Step Functions Standard workflow using a `.waitForTaskToken` task for approval
C. AWS Step Functions Express workflow
D. SQS queues between each step with a cron job on EC2 checking approvals

**Q10.** A company is migrating an on-premises application that exchanges messages through Apache ActiveMQ using JMS. The company wants a managed service and must avoid changing application code. What should the architect use?
A. Amazon SQS
B. Amazon SNS
C. Amazon MQ
D. Amazon Kinesis Data Streams

<details>
<summary><strong>Answers</strong></summary>

1. **A, C** — The queue buffers the spike so no orders are lost; scaling on backlog per instance adds workers as the queue grows. A load balancer (B) still delivers synchronously; SNS over HTTP (E) pushes and doesn't buffer.
2. **B** — FIFO gives ordering and exactly-once processing within a message group; using account ID as the group keeps per-account order while different accounts run in parallel. Standard (A) can reorder and duplicate.
3. **B** — S3 → SNS → per-processor SQS is the fan-out pattern: each queue buffers and scales independently, and adding a processor is just a subscription. S3 can't send the same event type for overlapping prefixes to multiple destinations (A).
4. **C** — Multiple independent real-time consumers plus replay = Kinesis Data Streams with extended retention. SQS (A) deletes messages once consumed; Firehose (D) delivers, it doesn't support independent readers or replay.
5. **B** — Firehose converts JSON to Parquet (with a Glue schema) and partitions S3 prefixes with no servers. A and D need you to run consumers.
6. **C** — Lambda ENIs in a VPC never get public IPs, so they need a NAT gateway (in a public subnet) for internet access. An IGW route from private subnets (A) doesn't work without public IPs; B isn't possible.
7. **B** — RDS Proxy pools and reuses connections across many concurrent functions. Provisioned concurrency (D) would, if anything, keep more connections open.
8. **B** — Lambda's maximum is 15 minutes (A is impossible). Fargate runs the custom container serverlessly, and EventBridge can launch it directly from the S3 event.
9. **B** — Standard workflows run up to a year with exactly-once execution and full history; task tokens pause for human approval; Retry handles backoff. Express (C) is limited to 5 minutes.
10. **C** — Amazon MQ is managed ActiveMQ/RabbitMQ with native JMS support, so code stays the same. SQS (A) would need code changes to its API.

</details>

---

## 13. Flashcards (cover the right column)

| Prompt | Answer |
| --- | --- |
| SQS default / max visibility timeout | 30 seconds / 12 hours |
| SQS retention range | 1 minute – 14 days (default 4 days) |
| Max long-polling wait | 20 seconds |
| Max SQS delay | 15 minutes |
| SQS Standard delivery guarantee | At-least-once, best-effort ordering |
| FIFO ordering scope | Per message group ID |
| What moves poison messages aside? | DLQ via redrive policy (`maxReceiveCount`) |
| Recommended SQS visibility timeout for a Lambda consumer | ≥ 6 × function timeout |
| SNS → SQS not delivering with SSE-KMS | KMS key policy must allow SNS |
| Filter what each SNS subscriber receives | Subscription filter policy |
| Serverless cron | EventBridge Scheduler |
| Reprocess old events on a bus | EventBridge archive & replay |
| Kinesis Data Streams default / max retention | 24 hours / 365 days |
| Kinesis ordering scope | Per shard (same partition key → same shard) |
| Dedicated 2 MB/s per consumer per shard | Enhanced fan-out |
| Firehose: real-time or near real-time? | Near real-time (buffered) |
| Managed Kafka / managed ActiveMQ-RabbitMQ | MSK / Amazon MQ |
| Lambda max timeout | 15 minutes |
| Lambda memory range | 128 MB – 10,240 MB (CPU scales with memory) |
| Remove cold starts | Provisioned concurrency (Java: SnapStart) |
| Guarantee and cap one function's concurrency | Reserved concurrency |
| Lambda in VPC → internet | Private subnet + NAT gateway |
| Lambda → RDS connection storms | RDS Proxy |
| Async Lambda failure handling | Retries, then on-failure destination (or DLQ) |
| API Gateway type with caching, usage plans, WAF, private endpoints | REST API |
| API Gateway default integration timeout | 29 seconds |
| Are API keys authentication? | No — usage plans/metering only |
| API Gateway → SQS without Lambda | AWS service integration |
| Step Functions Standard vs Express max duration | 1 year vs 5 minutes |
| Pause a workflow for human approval | `.waitForTaskToken` callback |

## 14. Before you close today

- [ ] Read sections 1–10 (focus: SQS lifecycle, Lambda invocation models, the KDS vs Firehose vs SQS table)
- [ ] Lab done and torn down (queues, topic, function, log group, role, any schedule)
- [ ] 10 questions above + 20 bank questions on SQS/SNS/Kinesis/Lambda/API Gateway/Step Functions
- [ ] Misses logged in `miss-log.md` — and add any Day 7 practice-exam misses you haven't logged yet

**Tomorrow (Day 9):** Containers and the edge — ECS vs EKS vs Fargate, Route 53 routing policies and health checks, CloudFront (origins, OAC, caching, Lambda@Edge vs CloudFront Functions) and Global Accelerator.
