# Firestore

Firestore is GCP's serverless document database for operational workloads. It stores JSON-like documents, supports real-time updates, and scales automatically.

## Use Cases
- Store application state or metadata needed by pipelines (configs, checkpoints, user state).
- Build event-driven apps that react to changes via triggers.
- Provide low-latency reads for dashboards, APIs, and mobile/web apps.
- Avoid managing database servers and capacity planning.

## Mental Model
- Data lives in documents grouped into collections.
- Documents are small, indexed, and read via queries or direct keys.
- Real-time listeners stream changes as they happen.
- Costs are per read/write/delete and per stored data.

## Quick Mental Mapping Table
| Relational Concept | Firestore Equivalent |
| ------------------ | -------------------- |
| Database           | Project              |
| Table              | Kind / Collection    |
| Row                | Entity / Document    |
| Column             | Field / Property     |
| Index              | Index                |

## Core Concepts
- Document: JSON-like record (max 1 MiB) with fields.
- Collection: group of documents.
- Subcollection: collection nested under a document.
- Index: required for most queries; composite indexes for multi-field filters.
- Query: filters, ordering, and limits over a collection.

## Data Modeling Patterns
- Denormalize for reads: duplicate fields to avoid joins.
- Use map fields for sparse attributes; arrays for small lists.
- Prefer fan-out writes over deep, unbounded nesting.
- Keep hot documents small to reduce contention.

## Queries And Indexing

**Single-field indexes — automatic:**
- Every field gets an **ascending** and **descending** index automatically (exam term: "atomic values, ascending/descending").
- No manual setup needed for simple equality or range queries on one field.

**Composite indexes — manual:**
- Multi-field filters or mixed sort orders require an explicit composite index.
- Firestore won't run the query without it — it fails at runtime.

| Index Type | Created By | Covers |
| --- | --- | --- |
| Single-field (atomic, asc + desc) | Automatic | One-field filters and sorts |
| Composite | Manual | Multi-field filters or mixed ordering |

- Queries need indexes that match the filter and sort order exactly.
- Inequality filters (`<`, `>`, `!=`) require an index and constrain sort order.
- Collection group queries scan all subcollections sharing the same name.

>"Composite, single value" is not a valid Firestore index type. Automatic indexes are **single-field only** — ascending and descending. Any multi-field combination requires a manually created composite index.

## Transactions And Consistency
- Strong consistency for document reads in the same region.
- Transactions are optimistic and retry on conflicts.
- Batch writes group up to 500 operations.
- Design for idempotency when retried.

## Scaling And Limits
- Horizontal scaling is automatic.
- Write throughput is limited per document; spread writes across keys.
- Use sharded counters for high-write aggregates.
- Keep document sizes below limits for stable latency.

## Security And Access Control
- Use [[Security/IAM|IAM]] for admin access and service accounts.
- Firestore Security Rules protect client access.
- Prefer service-to-service access for pipelines.

## Integrations
- Cloud Functions: trigger on document create/update/delete.
- Cloud Run: read/write operational state for services.
- [[Processing/Dataflow|Dataflow]]: sync operational state or metadata.
- [[Storage/BigQuery|BigQuery]]: export for analytics (via export or Dataflow).

## Monitoring And Ops
- Track read/write rates and latency.
- Alert on spikes in reads or writes (cost risk).
- Watch index build times and failures.

## Common Pitfalls
- Assuming SQL joins or complex aggregations.
- Unindexed queries failing at runtime.
- Hot document writes causing contention.
- Overlooking cost impact of frequent reads.

## Quick Checklist
- Define collections, document keys, and access patterns.
- Create required composite indexes before launch.
- Set Security Rules and test them.
- Plan for idempotent writes and retries.
- Monitor costs and query patterns early.
