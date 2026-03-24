# Pub/Sub

Pub/Sub is GCP's managed messaging service for event-driven systems. It delivers messages from publishers to subscribers with durable storage, horizontal scaling, and simple fan-out.

## Use Cases
- Ingest real-time events for streaming pipelines (often into [[Processing/Dataflow|Dataflow]] and [[Storage/BigQuery|BigQuery]]).
- Decouple producers and consumers to avoid tight dependencies.
- Fan out the same event stream to multiple consumers (analytics, monitoring, ML).
- Buffer bursts of events without dropping data.

## Mental Model
- Topics are append-only event streams; subscriptions are independent views.
- Pull subscriptions: subscribers poll and ack messages.
- Push subscriptions: Pub/Sub delivers to an HTTPS endpoint.
- Delivery is **at-least-once**, so consumers must be idempotent.
- Message retention and ack state are per subscription (not per topic); a message is deleted only after that subscription ACKs.

## Core Concepts

| Concept      | Description                                               |
| ------------ | --------------------------------------------------------- |
| Topic        | Named stream of messages                                  |
| Subscription | Cursor + delivery configuration for a topic               |
| Message      | Payload + attributes (key/value metadata)                 |
| Ack          | Confirms successful processing; unacked messages retried  |
| Retention    | How long unacked and acked messages are kept              |

## Delivery And Ordering
- At-least-once delivery: each published message is delivered one or more times; duplicates can happen.
- Duplicates occur when a subscriber receives a message but fails to ack before the deadline or hits a transient error, so Pub/Sub redelivers.
- Pub/Sub is not at-most-once; it prioritizes durability over uniqueness, so loss is avoided at the cost of duplicates.
- Message ordering is not guaranteed unless ordering keys are used.
- Ordering keys preserve order per key, but can reduce parallelism.

## Pull vs Push

**Pull:**
- Best for controlled scaling and backpressure.
- Subscribers manage their own parallelism and acking.

**Push:**
- Pub/Sub calls your endpoint; simpler for lightweight consumers.
- Requires stable HTTPS endpoint and careful retry handling.

## Exactly-Once And Idempotency
Pub/Sub supports exactly-once delivery for pull subscriptions with compatible clients.
- Still design consumers to be idempotent (safe to reprocess).
- Common pattern: write with upsert keys or dedupe in [[Storage/BigQuery|BigQuery]].

## Dead Letter Topics (DLQ)
Use a dead letter topic to isolate poison messages.
- Configure max delivery attempts on the subscription.
- Route failures to a DLQ for later inspection and reprocessing.
- Use a **separate topic** for the DLQ — routing back to the same topic causes infinite retry loops.

**Pub/Sub DLQ vs Dataflow side outputs:**
- **Pub/Sub DLQ**: handles subscriber/ack failures — message exceeds max delivery attempts → routed to the DLQ topic.
- **Dataflow side outputs**: handles transform errors — records that fail processing logic are routed to a dead-letter `PCollection`, not the Pub/Sub DLQ. These are separate mechanisms, not interchangeable.

## Schema And Validation
Pub/Sub can enforce schemas at publish time — messages that violate the schema are **rejected before entering the topic**.
- This is producer-side validation, not a consumer-side transformation.
- Define a schema, attach it to the topic, and require validation on publish.
- Assign schemas at the topic level so every subscription is automatically protected.

**Supported Schema Types:**

| Type         | Best When                                                          |
| ------------ | ------------------------------------------------------------------ |
| **Protobuf** | Strongly typed services, gRPC systems, minimal wire size           |
| **Avro**     | Analytics pipelines, Dataflow/BigQuery consumers, schema evolution |

**Unsupported Formats (Exam Traps):**

| Format  | Why It Fails                                                     |
| ------- | ---------------------------------------------------------------- |
| Thrift  | Similar to Protobuf conceptually, but not implemented in Pub/Sub |
| CSV     | No type system — cannot enforce field-level validation           |
| Parquet | Storage/analytics format, not a messaging schema standard        |

## Performance And Throughput
- Scale by increasing subscriber concurrency and ack throughput.
- Keep messages small and avoid huge payloads; store large blobs in [[Cloud-Storage|Cloud Storage]] and send references.
- For high-volume streams, use batching in publishers to improve efficiency.

## Security And Access Control
- Use [[Security/IAM|IAM]] roles: publisher, subscriber, and viewer.
- Prefer service accounts; avoid user credentials in long-running services.
- Use CMEK when required for customer-managed encryption keys (via [[Cloud-KMS|Cloud KMS]]).
- Use VPC Service Controls perimeters to prevent cross-project data exfiltration.

## Monitoring And Ops
- Key metrics: backlog size, oldest unacked message age, ack rate.
- Alert on growing backlog or increasing message age.
- Use retry policies and DLQs to avoid stuck pipelines.

**Subscription lag signals:**
- `subscription/num_undelivered_messages` — backlog (undelivered + unacked messages).
- `subscription/oldest_unacknowledged_message_age` — max wait time; rising values mean consumers are falling behind.

**Outage recovery:**
- Prefer **Pub/Sub Seek** (replay within retention window) over restarting Dataflow or reloading from Cloud Storage.
- Rule: seek window ≥ RPO + detection/restore lag; expect duplicates → keep sinks idempotent.

**Bad ACK recovery:**
- Create a **subscription snapshot** before risky changes; **Seek** back if needed.
- ACKed messages are not retained — expect duplicates; keep sinks idempotent.

## Common Pitfalls
- Forgetting idempotency on consumers — at-least-once delivery means duplicates *will* happen under retries and failures; use upsert keys, deduplication in [[Storage/BigQuery|BigQuery]], or exactly-once pull subscriptions.
- Missing DLQ configuration — without a DLQ, repeatedly failing messages consume delivery attempts until dropped silently; configure max delivery attempts and route failures to a dedicated DLQ topic.
- Routing the DLQ back to the same topic — creates an infinite retry loop as failed messages are re-enqueued and fail again; always use a **separate** DLQ topic.
- ACK deadline shorter than consumer processing time — Pub/Sub redelivers if no ACK arrives before the deadline, causing duplicate processing; extend the deadline or call `modifyAckDeadline` during long operations.
- Misaligned regions with downstream services — cross-region delivery adds latency and egress cost; co-locate topics and subscriptions with Dataflow, BigQuery, and other processing resources.
- Overusing ordering keys — all messages for a key are routed to a single subscriber, reducing throughput; only enable ordering when strict per-key sequence is genuinely required.
- Sending large payloads directly — Pub/Sub has a 10 MB per-message limit; store large objects in [[Cloud-Storage|Cloud Storage]] and publish a reference URI in the message instead.

## Integrations
- [[Processing/Dataflow|Dataflow]]: primary streaming processing engine for Pub/Sub.
- [[Storage/BigQuery|BigQuery]]: common sink for event analytics (streaming or batch).
- [[Cloud-Storage|Cloud Storage]]: store large payloads and reference them in messages.
- Cloud Functions / Cloud Run: event-driven compute consumers. Deploy a function with a Pub/Sub trigger using:
  ```bash
  gcloud functions deploy FUNCTION_NAME \
    --runtime=RUNTIME \
    --trigger-topic=TOPIC_NAME
  ```

## Quick Checklist
- Define topic purpose, schema, and retention policy.
- Choose pull vs push subscriptions based on consumer control needs.
- Decide on ordering keys and accept the throughput tradeoff.
- Configure DLQ and max delivery attempts.
- Ensure consumers are idempotent and observable.
