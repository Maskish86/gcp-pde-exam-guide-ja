# Cloud SQL

Cloud SQL is GCP's managed relational database service for MySQL, PostgreSQL, and SQL Server. It handles provisioning, patching, backups, and high availability while you manage schemas, users, and queries.

## Use Cases
- Metadata stores and small relational apps that support pipelines.
- Source systems for CDC into analytics (via [[Ingestion/Datastream|Datastream]]).
- Reference/lookup tables for ETL jobs or feature pipelines.

## Mental Model
- Google manages the database infrastructure; you manage database objects and access.
- Storage is persistent; compute scales vertically per instance.
- High availability uses a primary + standby with automated failover.

## Core Concepts

| Concept              | Description                                  |
| -------------------- | -------------------------------------------- |
| Instance             | The database server (region + machine type)  |
| Database/user        | Logical database and credentials             |
| Connectivity         | Private IP (VPC) or public IP                |
| Authorized networks  | IP allowlist for public IP connections       |
| Flags                | Engine-specific configuration                |

## Connectivity And Security

**Private IP:**
- Best for stable, internal networks (VPC and peering).
- Avoids public exposure and IP allowlists.

**Public IP:**
- Use when clients are outside VPC or on the internet.
- Requires strong auth and secure transport.

**Auth Proxy / Connectors:**
- Recommended for dynamic IP clients and production workloads.
- App connects locally to the proxy → proxy authenticates via service account / Application Default Credentials → proxy opens a TLS-encrypted tunnel to Cloud SQL.
- Handles TLS automatically — no manual SSL certificate management, no public IP exposure needed.
- Uses short-lived OAuth2 tokens; Authorized networks can be left empty.
- Avoid `0.0.0.0/0` allowlists for production.

> IAM (`roles/cloudsql.client`) controls *who* can connect, but does not establish the secure channel. The Auth Proxy handles the authentication flow, token exchange, and TLS — IAM and the proxy work together.

## Backups And Recovery
- Automated backups and point-in-time recovery.
- Test restore procedures; keep retention aligned with compliance.

## Performance And Scaling
- Right-size machine type and storage.
- Use read replicas for read-heavy workloads.
- Use connection pooling to avoid exhausting connections.

## Common Pitfalls
- Using public IP allowlists for dynamic clients — IPs change and allowlists break silently; use Auth Proxy or Cloud SQL Connector instead.
- Confusing HA standby with a read replica — the standby handles failover only, it does not serve reads; add a read replica for read scaling.
- Connection exhaustion from serverless workloads — Cloud Run/Functions can spawn hundreds of connections per cold-start burst; use connection pooling (PgBouncer for PostgreSQL, or Auth Proxy pool settings).
- Storage auto-increase is irreversible — the disk grows automatically but cannot shrink; oversized instances require creating a new instance and migrating data.
- Skipping backup restore tests — automated backups run on schedule, but PITR paths and restore procedures are rarely validated until an incident.
- Missing Datastream prerequisites for CDC — MySQL requires binary logging enabled; PostgreSQL requires `wal_level=logical`; forgetting these flags blocks Datastream setup entirely.

## Integrations
- [[Ingestion/Datastream|Datastream]]: CDC from Cloud SQL into analytics.
- [[Processing/Dataflow|Dataflow]]: ETL and enrichment.
- [[Security/IAM|IAM]]: IAM roles for proxy/connectors.

## Quick Checklist
- Decide private vs public IP connectivity.
- Use Auth proxy/connector for dynamic IP clients.
- Grant least-privilege IAM and database users.
- Configure backups and test restore.
- Enable monitoring and alerts.
