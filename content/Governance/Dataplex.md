# Dataplex

Dataplex is GCP's data fabric service that unifies data management across [[Cloud-Storage|Cloud-Storage]] and [[Storage/BigQuery|BigQuery]]. It provides centralized governance, metadata, and data quality for lakes and warehouses without moving data.

## Use Cases
- Apply consistent governance and metadata across multiple data lakes and warehouses.
- Catalog and classify data assets for discovery and compliance.
- Standardize data quality rules and monitoring.
- Organize data domains with clear ownership and access patterns.

## Mental Model
- Dataplex does not store data; it manages it in-place.
- You register data locations and Dataplex layers governance on top.
- Dataplex can auto-discover/catalog data in raw zones and apply governance/quality signals for curated-zone pipelines; transformations belong to [[Processing/Dataflow|Dataflow]] / [[Processing/Dataproc|Dataproc]] / [[Storage/BigQuery|BigQuery]].
- Assets are tied to lakes and zones, not to specific pipelines.
- It complements [[Storage/BigQuery|BigQuery]] and [[Cloud-Storage|Cloud Storage]], not replaces them.

## Core Concepts
- Lake: top-level container for data domains.
- Zone: logical grouping (raw, curated, sandbox) within a lake.
- Asset: a data resource ([[Cloud-Storage|GCS]] bucket or [[Storage/BigQuery|BigQuery]] dataset) registered in a zone.
- Entity: a table or file-based dataset recognized by the catalog.
- Catalog: metadata layer (schemas, tags, classifications).

## Governance And Catalog
- Centralized metadata for assets and entities.
- Tag templates for classifications (PII, sensitivity, owner).
- Access policies and discovery controls.
- Lineage integration where available (via [[Data-Catalog|Data Catalog]]).
- Dataplex enforces governance across [[Cloud-Storage|GCS]] and [[Storage/BigQuery|BigQuery]], while [[Data-Catalog|Data Catalog]] is discovery/metadata only.

## Lineage And Discovery (Key Capabilities)
- Unified discovery across projects and services with searchable metadata.
- Automatic lineage where supported for end-to-end visibility.
- Built-in data quality scans and profiling for quick, managed validation.

## Data Quality
- Define rules (null checks, ranges, schema constraints).
- Schedule scans and track quality over time.
- Use scorecards to surface data health.

## Security And Access Control
- [[Security/IAM|IAM]]-based access to lakes, zones, and assets.
- Inherits underlying storage access ([[Cloud-Storage|GCS]]/[[Storage/BigQuery|BigQuery]]).
- Use policy tags with [[Storage/BigQuery|BigQuery]] for column-level security.

## Operations And Monitoring
- Monitor scans, jobs, and data quality results.
- Track asset changes and schema drift.
- Use alerts for failing quality rules.

## Common Pitfalls
- Registering assets without clear ownership — unowned assets become ungoverned; no one is responsible for quality, access reviews, or compliance; assign ownership at registration and enforce it as a policy.
- Assuming Dataplex enforces access independently of IAM — Dataplex layers governance on top of existing IAM; access to underlying GCS buckets and BigQuery datasets is still controlled by IAM directly; Dataplex policies alone do not restrict data reads or writes.
- Skipping zone design and mixing raw and curated data — without raw/curated separation, unvalidated data contaminates downstream consumers; zone structure is what makes data quality rules and lineage meaningful.
- Treating Dataplex as a transformation or processing tool — Dataplex discovers, catalogs, and governs data in-place; it does not run transforms; processing belongs to [[Processing/Dataflow|Dataflow]], [[Processing/Dataproc|Dataproc]], or [[Storage/BigQuery|BigQuery]].
- Defining quality rules without alerting on failures — scans run on schedule and record results, but nothing is acted on without alerts; set up notifications on failing quality rules so issues reach the responsible owner automatically.
- Confusing Dataplex with [[Governance/Data-Catalog|Data Catalog]] — Data Catalog handles discovery, tagging, and metadata search for individual assets; Dataplex provides lake-level governance, zone management, data quality scans, and unified policy across a full data domain.

## Integrations
- [[Cloud-Storage|Cloud-Storage]]: lake storage assets.
- [[Storage/BigQuery|BigQuery]]: warehouse datasets as assets.
- [[Processing/Dataflow|Dataflow]] / [[Processing/Dataproc|Dataproc]]: processing pipelines that write into Dataplex-managed zones.
- [[Security/IAM|IAM]]: access control foundation.

## BigLake 
BigLake provides a unified storage layer for data in [[Cloud-Storage|Cloud Storage]] and [[Storage/BigQuery|BigQuery]] with consistent access control and governance. It lets you use BigQuery-style permissions and fine-grained access across lake and warehouse data without moving it.

## Quick Checklist
- Define lake and zones (raw/curated/sandbox).
- Register assets ([[Cloud-Storage|GCS]] buckets, [[Storage/BigQuery|BigQuery]] datasets).
- Apply tags/classifications and ownership metadata.
- Set data quality rules and monitoring.
- Align [[Security/IAM|IAM]] with underlying storage access.
