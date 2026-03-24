# AlloyDB

AlloyDB is GCP's managed, PostgreSQL-compatible database optimized for high performance and analytics-friendly workloads. It is designed for operational workloads that need better throughput and lower latency than standard PostgreSQL, while remaining compatible with PostgreSQL tools and drivers.

## Use Cases
- OLTP systems that need high read/write throughput.
- Choose **AlloyDB over Cloud SQL for PostgreSQL** when you need **higher throughput** or **HTAP** (mixed OLTP + analytics) without re‑platforming.
- Hybrid workloads with frequent analytics queries on operational data.
- Applications that want PostgreSQL compatibility with managed scaling.
- Source system for CDC into analytics (via [[Ingestion/Datastream|Datastream]]).

## Mental Model
- PostgreSQL-compatible engine with separated compute and storage.
- **HTAP = OLTP + analytics in one system** (row engine + columnar accelerator), reducing ETL/dual‑store complexity.
- Primary + read pool for scaling reads and analytics queries.
- Storage is distributed and managed; compute can scale independently.
- Compatible with most PostgreSQL extensions and drivers.

## Core Concepts
- Cluster: primary instance plus optional read pool.
- Primary: handles writes and transactions.
- Read pool: read-only nodes for scaling queries.
- Backups: automated with point-in-time recovery.

## Performance And Scaling
- Scale reads by adding read pool nodes.
- Scale compute vertically for write-heavy workloads.
- Keep hot data in memory; watch cache hit rates.

## Availability And Reliability
- High availability with automated failover.
- Regional deployment; use replicas for resilience.
- Backups and PITR for recovery.

## Security And Access Control
- IAM for resource access and management.
- Use private IP for internal access where possible.
- CMEK supported via [[Cloud-KMS|Cloud KMS]] for encryption control.

## Common Pitfalls
- Treating it like a warehouse; it is still an OLTP database.
- Under-sizing primary compute for write-heavy workloads.
- Not separating analytical reads to a read pool.

## Integrations
- [[Ingestion/Datastream|Datastream]]: CDC from AlloyDB into analytics.
- [[Processing/Dataflow|Dataflow]]: ETL from AlloyDB to [[Storage/BigQuery|BigQuery]].
- [[Security/IAM|IAM]] and [[Cloud-KMS|Cloud KMS]]: access and encryption controls.

## Quick Checklist
- Decide primary size and read pool count.
- Choose private IP or secure connectivity.
- Configure backups and test PITR.
- Set IAM and CMEK if required.
- Plan CDC if analytics replication is needed.
