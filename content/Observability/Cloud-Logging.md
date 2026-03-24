# Cloud Logging

Cloud Logging collects, stores, and analyzes log data from GCP services and your applications. It powers searchable logs, log-based metrics, and routing to sinks for long-term retention or analytics.

## Use Cases
- Debug failed pipelines and job errors.
- Trace data freshness issues via task logs.
- Build log-based metrics for alerting.
- Route logs to storage or analysis systems for compliance.

## Mental Model
- Logs are ingested into Log Explorer and stored per project.
- Logs can be routed to sinks (BigQuery, Cloud Storage, Pub/Sub) for retention or analysis.
- Log-based metrics turn log patterns into time series for alerting.
- Log severity and labels help filter noisy streams.

## Core Concepts
- Log entry: a single event with timestamp, severity, and payload.
- Log bucket: storage for logs within a project or org.
- Log sink: export route to another destination.
- Log-based metric: metric generated from log filters.

## Common Patterns
- Export Dataflow job logs to BigQuery for analysis.
- Create alerts on error counts from critical DAGs.
- Use structured logging for pipeline steps and row counts.
- Centralize logs across projects with aggregated sinks.

## Integrations
- [[Cloud-Monitoring|Cloud-Monitoring]]: alert on log-based metrics.
- [[Processing/Dataflow|Dataflow]]: job and worker logs.
- [[Processing/Dataproc|Dataproc]]: cluster and job logs.
- [[Storage/BigQuery|BigQuery]]: log export destination.
- [[Cloud-Storage|Cloud Storage]]: log archive destination.
- [[Ingestion/PubSub|Pub/Sub]]: stream logs to downstream consumers.

## Security And Access Control
- Use least-privilege [[Security/IAM|IAM]] roles for log access.
- Limit access to sensitive logs containing PII or secrets.
- Use log sinks to separate retention by sensitivity.

## Operations And Reliability
- Standardize log formats for pipeline steps.
- Retain logs long enough for audits and postmortems.
- Use labels to track environment and pipeline version.

## Common Pitfalls
- Logging sensitive data without masking.
- Relying on logs for metrics instead of emitting explicit metrics.
- High log volume causing cost surprises without sinks or filters.

## Quick Checklist
- Define log retention by environment and compliance needs.
- Create sinks for long-term storage or analysis.
- Add log-based metrics for critical error patterns.
- Document log search filters for on-call use.
- Review log volume and costs quarterly.
