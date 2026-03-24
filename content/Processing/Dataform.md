# Dataform

Dataform is a data transformation framework for [[Storage/BigQuery|BigQuery]]. It lets you define SQL-based pipelines, manage dependencies, and run incremental builds with version control.

## Use Cases
- Build curated datasets from raw/staging tables.
- Manage SQL transformations with dependency graphs.
- Implement incremental models with repeatable runs.
- Share reusable data models with governed practices.

## Mental Model
- Dataform compiles SQL into a DAG of operations.
- Runs are orchestration of SQL in BigQuery, not separate compute.
- Incremental tables avoid full rebuilds when designed correctly.
- Environments (dev/prod) should separate datasets and permissions.

## Core Concepts
- Project: a collection of models and configs.
- Model: a SQL transformation (table, view, incremental).
- Assertions: data quality checks (uniqueness, nulls, constraints).
- Dependencies: explicit refs between models.
- Runs: execution of a compiled graph in BigQuery.

## Common Patterns
- Raw -> staging -> curated layering with clear naming.
- Incremental models using `uniqueKey` and `updatePartitionFilter`.
- Assertions for critical business rules and freshness.
- Version control with pull requests and code review.

## Assertions For Uniqueness And Nulls
- Use built-in assertions (`uniqueKey`, `nonNull`, or custom) for data quality checks.
- Assertions run as part of the Dataform pipeline and fail the run on violations.
- This is the most efficient, native approach compared to external DQ tools or manual UDF checks.
## Integrations
- [[Storage/BigQuery|BigQuery]]: execution engine and storage.
- [[../Security/IAM|IAM]]: permissions for service accounts and users.
- [[Cloud-Composer|Cloud-Composer]] or [[../Orchestration/Workflows|Workflows]]: schedule and trigger runs.

## Security And Access Control
- Use least-privilege [[../Security/IAM|IAM]] for Dataform service accounts.
- Separate dev/prod datasets to prevent accidental writes.
- Avoid embedding secrets; use managed connections where possible.

## Operations And Reliability
- Use incremental builds for large tables to control cost.
- Monitor failures and set alerts on scheduled runs.
- Keep SQL style consistent to improve maintainability.

## Common Pitfalls
- Using incremental models without a stable unique key.
- Mixing environments in one dataset.
- Ignoring assertions until after a failure.

## Quick Checklist
- Define dataset naming conventions (raw/staging/curated).
- Set up environments and permissions.
- Add assertions for critical tables.
- Implement incremental logic where appropriate.
- Document run schedules and ownership.
