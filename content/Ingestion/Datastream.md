# Datastream

Datastream is GCP's managed change data capture (CDC) service. It captures row-level changes from source databases and streams them to destinations like [[Storage/BigQuery|BigQuery]] or [[Cloud-Storage]] with minimal operational overhead.

## Use Cases
- Replicate OLTP data into analytics systems with low latency.
- Build CDC pipelines without managing Debezium or custom log readers.
- Keep a historical trail for auditing or replay.
- Feed streaming pipelines (often [[Processing/Dataflow|Dataflow]]) for transformation or routing.
- Use a fully managed service without Kafka or custom CDC clusters.

## Mental Model
- Datastream reads database logs (not table scans) for ongoing changes.
- It emits ordered change events per table (insert/update/delete).
- Initial backfill is separate from continuous change capture.
- You usually land raw CDC events, then transform into curated tables.

## Core Concepts
- **Connection profile**: credentials and connectivity info for source/destination.
- **Stream**: defines sources, destinations, and options (objects, filters, backfill).
- **Backfill**: initial snapshot of selected tables.
- **Change stream**: continuous CDC events from logs.
- **Private connectivity**: secure network path to sources.

## Sources And Destinations

### Sources
- [[Cloud-SQL|Cloud SQL]] for MySQL/PostgreSQL.
- Self-managed MySQL/PostgreSQL (with proper connectivity).
- Oracle (CDC supported).

### Destinations
- [[Storage/BigQuery|BigQuery]] (CDC tables + metadata).
- [[Cloud-Storage|Cloud Storage]] (file-based CDC for downstream processing).

## Backfill And CDC Flow
Typical lifecycle:
1. Create connection profiles (source + destination).
2. Define a stream (select schemas/tables + backfill settings).
3. Run backfill to capture baseline tables.
4. Start continuous CDC to keep data current.

Backfill guidance:
- Use it for initial load; verify row counts and primary keys.
- Schedule in off-peak hours if source load is sensitive.

## Data Shape And Semantics
Datastream emits changes, not final state:
- Inserts/updates/deletes arrive as events with metadata (timestamp, source).
- You must apply changes to build a current-state table.
- Deletes are explicit events; downstream should handle tombstones.

## Target Patterns

### To [[Storage/BigQuery|BigQuery]]
- Datastream writes CDC tables; build curated tables with MERGE.
- Use event time from the CDC metadata for ordering.

### To [[Cloud-Storage|Cloud Storage]]
- CDC files can be processed by [[Processing/Dataflow|Dataflow]] or batch jobs.
- Good for replayable, append-only pipelines.

## Performance And Cost
- Scope streams to required schemas/tables to reduce load and cost.
- Monitor lag between source and destination.
- Large backfills can be expensive; validate data volumes first.

## Reliability And Monitoring
- Track stream health, lag, and error logs.
- Ensure source log retention exceeds stream lag to avoid gaps.
- Reconcile with periodic row counts or checksums.

## Security And Governance
- Use least-privilege database users for CDC.
 - When the source DB (e.g., **Oracle**) is in a VPC, choose private connectivity (PSC/VPC peering) to avoid public IP exposure.
- Use CMEK where required for destinations (via [[Cloud-KMS|Cloud KMS]]).

## Common Pitfalls
- Missing primary keys (hard to upsert cleanly downstream).
- Insufficient source log retention causing data loss.
- Assuming CDC tables are "final" without applying changes.
- Region mismatch between sources and destinations.

## Quick Checklist
- Verify source database log settings and retention.
- Decide destination ([[Storage/BigQuery|BigQuery]] vs [[Cloud-Storage|Cloud Storage]]) and downstream transform plan.
- Plan backfill timing and validate row counts.
- Ensure primary keys exist for reliable upserts.
- Set monitoring on lag and error rates.
