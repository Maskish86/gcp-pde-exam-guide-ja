# Data Catalog

Data Catalog is GCP's managed metadata and discovery service. It helps you find, describe, tag, and govern data assets across services like BigQuery and Cloud Storage without moving the data.

## Use Cases
- Create a searchable inventory of datasets, tables, and files.
- Apply business and technical metadata (owners, sensitivity, definitions).
- Standardize governance with tags and policy tags.
- Improve discoverability for analysts and downstream teams.

## Mental Model
- Data Catalog stores metadata only, not the data itself.
- Entries represent assets; tags add meaning and governance context.
- Policy tags drive column-level access control in BigQuery.
- It complements, not replaces, tools like [[Governance/Dataplex|Dataplex]].
- **Auto-extracts metadata** from [[Storage/BigQuery|BigQuery]], [[Cloud-Storage|Cloud Storage]], [[OperationalDBs/Bigtable|Bigtable]], and [[Ingestion/PubSub|Pub/Sub]] — no custom scripting needed.
- PostgreSQL on Compute Engine is not a managed metadata source, so Data Catalog cannot auto-scan it; use a connector.

## Core Concepts
- Entry: a data asset (table, dataset, file set, or custom entry).
- Tag template: schema for metadata fields (owner, PII, SLA, etc.).
- Tag: metadata instance attached to an entry.
- Policy tag: taxonomy for column-level security in BigQuery.

## Common Patterns
- Tag datasets with owners and SLAs to clarify accountability.
- Classify columns with policy tags for sensitive data controls.
- Use business tags to align assets with domains or products.
- Keep discovery current by auto-ingesting supported services.
- For Cloud Storage migrations, run [[Security/DLP|DLP]] to detect PII and then attach tags in Data Catalog to label sensitive objects.
- Store PII analysis results as structured tag metadata (e.g. `contains_pii: true`, `pii_type: [name, dob]`) so findings are queryable and retrievable later — this is a metadata storage problem, not a detection, anonymization, or encryption problem.

## Integrations
- [[Storage/BigQuery|BigQuery]]: tables, datasets, policy tags.
- [[Cloud-Storage|Cloud Storage]]: file set metadata for lakes.
- [[Governance/Dataplex|Dataplex]]: catalog and governance at scale.
- [[Security/IAM|IAM]]: governs who can view and edit metadata.

## Security And Access Control
- Use least-privilege [[Security/IAM|IAM]] for catalog access.
- Control tag visibility to limit sensitive metadata exposure.
- Use policy tags to enforce column-level security in BigQuery.
- Column security works only if policy tag access control is enabled and users lack `roles/datacatalog.categoryFineGrainedReader`.
- **tagTemplateViewer** is required to view non‑public tag templates; it’s redundant if the template is **public**.

## Operations And Reliability
- Keep tag templates consistent across domains.
- Review and prune stale tags to avoid confusion.
- Document ownership and definitions for critical assets.

## Common Pitfalls
- Leaving tags empty or skipping them entirely — untagged assets are invisible to governance and search; empty tags are noise; enforce shared tag templates across domains at rollout, not retroactively.
- Inconsistent tag schemas across teams — each team defining its own templates fragments search and makes cross-domain governance impossible; agree on a canonical template set before onboarding teams.
- Assuming catalog visibility grants data access — Data Catalog controls who sees and edits metadata; actual data access is governed separately by IAM on BigQuery, GCS, and other resources.
- Forgetting to restrict `categoryFineGrainedReader` for policy tags — policy tags only block column access if this role is withheld; granting it too broadly (e.g. to a large group or `allUsers`) silently defeats the column-level control.
- Expecting auto-scan for non-managed sources — Data Catalog auto-extracts from BigQuery, GCS, Bigtable, and Pub/Sub; custom or on-prem sources (PostgreSQL on Compute Engine, etc.) require a connector or manual entry.
- Not writing DLP scan results back as catalog tags — running DLP detects PII but does not populate Data Catalog automatically; explicitly write findings as structured tags (`contains_pii: true`, `pii_type`) so sensitive assets are discoverable and auditable.
- Confusing Data Catalog with [[Governance/Dataplex|Dataplex]] for lake-scale governance — Data Catalog handles discovery and tagging; Dataplex provides lake management, data quality rules, and unified governance across zones at scale.

## Quick Checklist
- Define tag templates for ownership, sensitivity, and SLAs.
- Classify sensitive columns with policy tags.
- Register key datasets and lake locations.
- Assign metadata owners and review cadence.
- Document how teams should discover and request access.
