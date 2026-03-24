# Looker

Looker is GCP's enterprise BI and semantic modeling platform. It defines a governed data model (LookML) on top of sources like BigQuery and serves trusted metrics to dashboards and embedded apps.

## Use Cases
- Provide a single source of truth for business metrics and dimensions.
- Prevent metric drift by centralizing definitions in LookML.
- Serve governed dashboards with row/column-level access controls.
- Embed analytics into internal apps with consistent logic.

## Mental Model
- LookML defines the semantic layer; SQL is generated at query time.
- Models separate raw tables from curated business views.
- Caching improves performance but can hide data freshness issues.
- Permissions are enforced at the semantic layer, not just the database.

## Core Concepts
- LookML: modeling language for dimensions, measures, joins, and explores.
- Model: a collection of explores and connections.
- Explore: a queryable view for end users.
- View: maps to tables or derived tables.
- Derived table: SQL-defined table (PDT/DT) for precomputation.
- Dashboard: saved visualizations on top of explores.

## Common Patterns
- Build LookML on curated tables from [[Storage/BigQuery|BigQuery]].
- Use derived tables for heavy transforms that are expensive at query time.
- Enforce row-level security based on user attributes.
- Version control LookML for review and rollback.

## Integrations
- [[Storage/BigQuery|BigQuery]]: primary warehouse backend.
- [[Cloud-Storage|Cloud Storage]]: data lake staging feeding BigQuery.
- [[Governance/Dataplex|Dataplex]]: governance and metadata alignment.
- [[Security/IAM|IAM]]: access control foundation for connections and users.

## Security And Access Control
- Use least-privilege connections to the warehouse.
- Align Looker permissions with [[Security/IAM|IAM]] groups.
- Apply row/column security in LookML to enforce data policies.

## Operations And Reliability
- Monitor dashboard load times and query costs in BigQuery.
- Use caching strategically; document freshness expectations.
- Keep LookML changes reviewed to avoid breaking downstream reports.

## Common Pitfalls
- Modeling on raw tables instead of curated datasets.
- Duplicating metric logic across explores.
- Ignoring query costs and cache invalidation behavior.

## Quick Checklist
- Define core metrics and dimensions in LookML.
- Point explores at curated BigQuery tables.
- Set row-level security rules and test access.
- Establish review and deployment workflow for LookML.
- Document ownership and data freshness expectations.
