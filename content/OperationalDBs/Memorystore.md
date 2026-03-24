# Memorystore

Memorystore is GCP's managed in-memory data store for Redis and Memcached. It provides sub-millisecond caching and ephemeral data storage without managing servers. **It is a cache, not a system of record** — data lives in memory and is not a substitute for durable storage.

## Use Cases
- Cache hot keys or expensive query results to reduce load on [[Storage/BigQuery|BigQuery]] or OLTP systems.
- Store session data, feature flags, or rate-limit counters.
- Speed up serving layers that need sub-millisecond reads.
- Enrich streaming pipelines ([[Processing/Dataflow|Dataflow]]) by serving lookup tables from cache.

## Mental Model
- It is a cache or ephemeral store, not a system of record.
- Data is stored in memory; persistence is limited and not a substitute for durable storage.
- Performance depends on key design, eviction policy, and memory sizing.
- Access is via **private IP (VPC peering only)** — no public endpoint.

## Core Concepts

| Concept         | Description                                                                             |
| --------------- | --------------------------------------------------------------------------------------- |
| Instance        | Managed Redis or Memcached deployment                                                   |
| Tier (Redis)    | Basic = single node, no replicas; Standard = primary + replicas with auto-failover      |
| Eviction policy | What happens when memory fills (LRU, volatile, noeviction, etc.)                        |
| Read replicas   | Redis Standard: read endpoint backed by replicas; shifts read fan-out from primary      |
| Failover        | Redis Standard: auto-promotes replica on zonal failure; Basic and Memcached: none       |
| AUTH / TLS      | Redis: supports password auth and in-transit encryption; Memcached: neither             |

## Redis vs Memcached

| Feature           | Redis                                          | Memcached                       |
| ----------------- | ---------------------------------------------- | ------------------------------- |
| Data structures   | Rich: lists, sets, sorted sets, hashes, streams | Simple key-value only           |
| Persistence       | Optional (RDB snapshots)                       | None                            |
| HA / Failover     | Standard tier only                             | None                            |
| Read replicas     | Standard tier only                             | None                            |
| Pub/Sub           | Yes                                            | No                              |
| Best for          | Feature-rich caching, sessions, leaderboards   | Simple, high-throughput caching |

- **Migration:** **Memorystore for Redis** supports **RDB (Redis Database) snapshot** → GCS → import (RDB = point‑in‑time disk snapshot of Redis data); **Memcached** has no persistence/import, so treat it as a cold cache (anti‑pattern: expecting stateful migration).

## Performance And Cost
- Memory sizing is the primary cost lever — provision with headroom for peak load.
- Monitor eviction rate; high evictions mean undersized memory or suboptimal key design.
- Standard tier adds cost for replicas; use Basic for dev/non-critical workloads.
- Read replicas (Standard) reduce primary load for read-heavy patterns.
- Eviction policy affects hit rate: `allkeys-lru` evicts any key; `volatile-lru` evicts only keys with a TTL; `noeviction` returns errors on full memory.

## Security And Governance
- Access via **VPC peering only** — Memorystore has no public IP.
- Enable **in-transit encryption (TLS)** for Redis (not supported by Memcached).
- Use **AUTH** (Redis password) to restrict data-plane access within the VPC.
- Use [[Security/IAM|IAM]] for resource management (create/delete instances); IAM does not control data-plane access.

## Common Pitfalls
- Using Memorystore as the only copy of critical data.
- Underprovisioning memory and triggering evictions.
- Ignoring regional placement and latency to clients.
- Choosing Memcached expecting HA — it has no replicas or failover.
- Using Basic tier for production workloads that require zero-downtime.

## Integrations
- [[Storage/BigQuery|BigQuery]]: cache expensive query results to reduce slot consumption.
- [[OperationalDBs/Cloud-SQL|Cloud SQL]] / [[OperationalDBs/Spanner|Spanner]]: cache database reads for low-latency serving.
- [[Processing/Dataflow|Dataflow]]: serve lookup/reference tables from cache for stream enrichment.
- Cloud Run / App Engine: session storage and rate-limit counters.

## Quick Checklist
- Choose Redis (feature-rich) vs Memcached (simple key-value only).
- Choose Basic (dev/testing) vs Standard (production, HA required).
- Size memory with headroom; define eviction policy based on use case.
- Enable TLS and AUTH for Redis in production.
- Place instance in the same region as clients.
- Monitor memory usage, eviction rate, and connection count.
