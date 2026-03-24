# Spanner

Spanner is GCP's globally distributed relational database with strong consistency at scale. It offers horizontal scaling, high availability, and multi-region replication for mission-critical OLTP workloads.

## Use Cases
- Global applications that need strong consistency across regions.
- OLTP systems that outgrow single-region databases.
- Large-scale transactional workloads with high availability requirements.
- Source system for CDC into analytics (via [[Ingestion/Datastream|Datastream]]).

## Mental Model
- Spanner scales horizontally via shards (splits).
- TrueTime provides globally synchronized timestamps for strong consistency.
- Multi-region configs provide low-latency reads and high availability.
- SQL schema with relational model, but with distributed design constraints.

## Core Concepts
- Instance: capacity allocation for databases.
- Database: schema and data stored in Spanner.
- Nodes/Processing Units: compute capacity for workload.
- Splits: key-range partitions managed by Spanner.
- TrueTime: time API enabling global consistency.

## Data Modeling Considerations
- Primary keys define data locality and performance.
- Avoid hot keys; distribute writes across key ranges.
- Use interleaved tables to co-locate related data.

## Hot-Spotting Key Point
- Hot-spotting occurs when the leading primary key is sequential (or time-ordered), so most new writes hit the same split.
- Key design goal: make the leading key component well-distributed across key ranges.

| Key Pattern                           | Choose When                             | Why It Works                             | Why the Wrong Choice Fails                                       |
| ------------------------------------- | --------------------------------------- | ---------------------------------------- | ---------------------------------------------------------------- |
| `PRIMARY KEY (hash_user_id, user_id)` | High write rate by entity id            | Hash prefix spreads writes across splits | `PRIMARY KEY (user_id)` or sequential ids can concentrate writes |
| `PRIMARY KEY (bit_reversed_id)`       | Need sequence-like ids without hotspots | Bit reversal scatters sequential inserts | Plain increasing sequence keeps appending to a hot split         |

- Common trap: using timestamp/auto-increment as the first key column in heavy-ingest tables.

**Interleaved tables:**

| Option | Use When | Why |
| --- | --- | --- |
| Interleaved child table | Child rows are usually read with parent rows | Co-locates child rows by parent key prefix, reducing cross-split joins |
| Non-interleaved child table | Parent and child are queried independently | More flexible, but parent-child joins can be distributed and slower |

- Rule: child primary key must start with the full parent primary key.
```sql
INTERLEAVE IN PARENT Orders ON DELETE CASCADE
```
## Performance And Scaling
- Add nodes for throughput and storage growth.
- Use read-only transactions and stale reads for lower latency when acceptable.
- Monitor splits and hotspots to guide key design.

## Availability And Consistency
- Strong consistency by default for reads and writes.
- Multi-region configurations for high availability.
- Regional configurations when low-latency single-region is enough.

## Security And Access Control
- IAM for instance and database access.
- CMEK supported via [[Cloud-KMS|Cloud KMS]] if required.

## Common Pitfalls
- Sequential or timestamp-based leading primary key — all new writes concentrate on the same split, creating a hotspot; use a hash prefix, bit-reversed ID, or any well-distributed leading key component.
- Assuming Spanner is a drop-in for single-node SQL — distributed design imposes constraints: no arbitrary cross-table joins, no native auto-increment sequences, and transactions carry latency cost; schema and query patterns must be redesigned.
- Under-provisioning for peak write traffic — splits take time to rebalance after scale-up; provision for peak load in advance and monitor hotspot metrics proactively.
- Using read-write transactions for read-only queries — read-write transactions acquire locks and add unnecessary latency; use read-only transactions or stale reads for queries that don't require the latest committed state.
- Running OLAP queries directly on Spanner — full-table scans and aggregations compete with OLTP traffic and spike latency; export to [[Storage/BigQuery|BigQuery]] via [[Ingestion/Datastream|Datastream]] or [[Processing/Dataflow|Dataflow]] for analytics workloads.
- Choosing multi-region when single-region suffices — multi-region configs provide global HA but cost significantly more; only choose multi-region when cross-region availability or global read latency is a hard requirement.

## Integrations
- [[Ingestion/Datastream|Datastream]]: CDC from Spanner to analytics.
- [[Processing/Dataflow|Dataflow]]: ETL to [[Storage/BigQuery|BigQuery]] or [[Cloud-Storage|Cloud Storage]].
- [[Security/IAM|IAM]] and [[Cloud-KMS|Cloud KMS]]: access and encryption controls.

## Quick Checklist
- Choose region or multi-region based on latency and HA needs.
- Design primary keys for write distribution.
- Size nodes/processing units for workload.
- Configure backups and monitoring.
- Plan CDC if analytics replication is needed.
