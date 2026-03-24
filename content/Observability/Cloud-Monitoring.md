# Cloud Monitoring

Cloud Monitoring (formerly Stackdriver) collects metrics and events from GCP services and your applications. It provides dashboards, alerts, and uptime checks so you can detect issues in data pipelines and infrastructure quickly.

## Use Cases
- Monitor pipeline latency, failures, and data freshness.
- Track service health (worker CPU, memory, disk, queue backlog).
- Alert on SLA/SLO violations for critical datasets and jobs.
- Build dashboards for ops visibility and incident response.

## Mental Model
- Metrics are time series tied to a monitored resource.
- Alerts evaluate metric conditions on a schedule and page when thresholds are breached.
- Dashboards visualize multiple related metrics for a single system.
- Uptime checks validate external endpoints and trigger alerts on failure.

## Core Concepts
- Metric: a single measured value (latency, errors, backlog).
- Monitored resource: the thing being measured (job, VM, service).
- Alert policy: rules for when to notify.
- Notification channel: email, SMS, Slack, PagerDuty, etc.
- Dashboard: curated views of related metrics.

## Common Patterns
- Alert on Pub/Sub oldest unacked message age (backlog risk).
- Track Dataflow watermark delay and throughput.
- Monitor Dataproc job failures and worker saturation.
- Use custom metrics for row counts or pipeline lag.

## Integrations
- [[Processing/Dataflow|Dataflow]]: job metrics and streaming lag.
- [[Processing/Dataproc|Dataproc]]: cluster and job health.
- [[Storage/BigQuery|BigQuery]]: query and slot metrics.
- [[Ingestion/PubSub|Pub/Sub]]: backlog size and delivery age.
- [[Cloud-Storage|Cloud Storage]]: request and latency metrics.
- [[Cloud-Logging|Cloud Logging]]: log-based metrics and alerting on errors.

## Security And Access Control
- Use least-privilege [[Security/IAM|IAM]] roles for dashboards and alerting.
- Separate viewers (read) from editors (write) to prevent alert sprawl.

## Operations And Reliability
- Name alert policies consistently by system and severity.
- Use multi-window alerting to reduce noise during brief spikes.
- Document runbooks in alert descriptions for faster triage.

## Common Pitfalls
- Alerting on low-signal metrics and generating noise.
- Missing per-project scoping for dashboards and alerts.
- Relying only on infrastructure metrics instead of pipeline outcomes.

## Quick Checklist
- Identify critical SLIs (latency, freshness, error rate).
- Create dashboards for core pipelines and dependencies.
- Add alert policies with clear thresholds and runbooks.
- Verify notification channels and on-call routing.
- Review alert noise and tune monthly.
