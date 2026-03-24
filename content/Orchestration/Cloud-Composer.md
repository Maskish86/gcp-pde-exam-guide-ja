# Cloud Composer

Cloud Composer is GCP's managed Apache Airflow service for orchestrating workflows. It schedules and runs DAGs that coordinate tasks across data systems, while Google manages the underlying infrastructure.

## Use Cases
- Schedule and monitor ELT/ETL pipelines across multiple GCP services.
- Coordinate dependencies (ingest → transform → publish) with retries and SLAs.
- Run batch workflows that need backfills and parameterised runs.
- Centralize operational visibility for multi-step data jobs.

## Mental Model
- A DAG defines task dependencies; the scheduler decides what runs next.
- Tasks should be idempotent — retries happen automatically on failure.
- Composer is **orchestration, not execution**: heavy compute stays in [[Processing/Dataflow|Dataflow]], [[Processing/Dataproc|Dataproc]], or [[Storage/BigQuery|BigQuery]].
- Environment health depends on DAG size, scheduling load, and worker capacity.

> "Multiple jobs with complex dependencies" → Cloud Composer. 
> "Simple API call chain or event-driven glue" → [[Orchestration/Workflows|Workflows]].

## Core Concepts

| Concept         | Description                                                                                                                      |                  |
| --------------- | -------------------------------------------------------------------------------------------------------------------------------- | ---------------- |
| **Environment** | Managed Airflow deployment — scheduler, web UI, workers, metadata DB                                                             |                  |
| **DAG**         | Directed acyclic graph of tasks (nodes) and dependencies (edges) defining order, scheduling/catchup, and parallelism (no cycles) |                  |
| **Task**        | Unit of work; one operator instance                                                                                              |                  |
| **Operator**    | Predefined task type (BigQuery, Dataflow, Bash, HTTP, etc.)                                                                      |                  |
| **Sensor**      | Waits for an external condition before proceeding (file, table, message)                                                         |                  |
| **Executor**    | Controls how tasks are distributed and scaled across workers                                                                     |                  |
| **Connection**  | Named credentials for external systems; store secrets in [[Security/Secret Manager                                               | Secret Manager]] |
| **REST API**    | Airflow API to trigger DAG runs, clear tasks, and fetch DAG/task status from external systems                                    |                  |

## Key GCP Operators

| Operator | Use |
| --- | --- |
| `BigQueryInsertJobOperator` | Submit BigQuery SQL jobs (scheduled queries, DML, DDL) |
| `DataflowCreateJobOperator` | Launch a Dataflow batch or streaming job |
| `DataprocSubmitJobOperator` | Submit a Spark/Hadoop job to a Dataproc cluster |
| `GCSObjectExistenceSensor` | Wait for a file to arrive in Cloud Storage before proceeding |
| `BigQueryTableExistenceSensor` | Wait for a table or partition to be populated |
| `PubSubPullSensor` | Wait for a Pub/Sub message |
| `BashOperator` | Run CLI commands (`bq`, `gcloud`, shell scripts) |

## Common DAG Patterns

**Ingest → stage → transform → publish:**
- Use sensors between steps to gate on data arrival before proceeding.
- Each task must be idempotent — use `WRITE_TRUNCATE`, `MERGE`, or partition overwrites.
- Prefer separate DAGs per table unless transformations are identical; avoid mega‑DAGs with heavy branching.

**Backfills:**
- `catchup=True` replays missed intervals automatically on DAG enable.
- Manual: `airflow dags backfill -s <start_date> -e <end_date> <dag_id>`
- Parameterise tasks by `{{ ds }}` (execution date) so each run targets the right partition.

**SLA monitoring:**
- Set `sla` on individual tasks; Airflow triggers `sla_miss_callback` when exceeded.
- Pair with email alerts or [[Ingestion/PubSub|Pub/Sub]] notifications for on-call escalation.

**Scheduled BigQuery with retries:**
- `BigQueryInsertJobOperator` with `retries=3`, `retry_delay=timedelta(minutes=5)`.
- Use `WRITE_APPEND` or `WRITE_TRUNCATE` based on idempotency requirements.
- Set `email_on_failure=True` for alerting after retries are exhausted.
- For 2-hour SQL with retries/alerts, prefer Composer + `BigQueryInsertJobOperator`; Workflows is too light for scheduling/backfills.

## Exam Decision: Composer vs Workflows

| Requirement signal                                                    | Choose                                 | Reject                                 | Why rejected                                                                                   |
| --------------------------------------------------------------------- | -------------------------------------- | -------------------------------------- | ---------------------------------------------------------------------------------------------- |
| Scheduled pipelines, backfills, SLAs, many cross-service dependencies | Cloud Composer                         | [[Orchestration/Workflows\|Workflows]] | Workflows is for lightweight API chaining; not built for DAG scheduling or backfill operations |
| Simple step orchestration (HTTP / GCP API calls), event-driven glue   | [[Orchestration/Workflows\|Workflows]] | Cloud Composer                         | Composer is heavier operationally; don't pick it for a few API calls with retries/branches     |

## Security And Access Control
- Least-privilege [[Security/IAM|IAM]] roles for the environment service account.
- Prefer private IP environments when possible.
- Store all credentials in [[Security/Secret-Manager|Secret Manager]] — not in DAG code or plaintext Airflow connections.
- Private Composer web server → use PSC (private endpoint) not public IP; public exposure violates constraint.

## Operations And Reliability
- Keep DAGs small; avoid heavy imports at parse time — slow parses delay the entire scheduler.
- Set sensible concurrency and parallelism limits to avoid worker exhaustion.
- Monitor task duration, queue time, and scheduler heartbeat.
- Use retries with exponential backoff for transient failures.
- Prefer Celery or Kubernetes executor over Sequential executor in production.

## Common Pitfalls
- Running heavy compute inside Airflow workers — workers are for orchestration only; offload to `DataflowCreateJobOperator`, `DataprocSubmitJobOperator`, or `BigQueryInsertJobOperator`.
- Heavy Python imports or dynamic DAG generation at parse time — the scheduler re-parses all DAG files on every cycle; slow parses delay the whole environment; keep module-level code minimal.
- Missing idempotency on retried tasks — Airflow reruns tasks automatically; writes must use `WRITE_TRUNCATE`, `MERGE`, or partition overwrites to avoid duplicating data.
- Sensors in `mode='poke'` with tight intervals — each poke holds a worker slot for its entire wait; use `mode='reschedule'` so the slot is released between checks.
- Enabling `catchup=True` without testing — replays every missed interval since `start_date`; unexpected enable can flood downstream systems; limit with `max_active_runs` and test on a short date range first.
- Storing credentials in DAG code or plaintext Airflow connections — secrets are visible in the metadata DB and web UI; store all credentials in [[Security/Secret-Manager|Secret Manager]] and reference them via the Secrets Backend.
- Using the Sequential executor in production — executes one task at a time, serializing the entire environment; use Celery or Kubernetes executor for any real workload.

## Integrations
- [[Storage/BigQuery|BigQuery]]: scheduled queries, dataset maintenance, and monitoring.
- [[Storage/Cloud-Storage|Cloud Storage]]: file arrival sensors and staging data.
- [[Processing/Dataflow|Dataflow]]: launch streaming or batch pipelines.
- [[Processing/Dataproc|Dataproc]]: submit Spark jobs for heavy processing.
- [[Ingestion/PubSub|Pub/Sub]]: event-driven triggers and notifications.
- [[Security/Secret-Manager|Secret Manager]]: store and retrieve connection secrets.

## Quick Checklist
- Choose region aligned with [[Storage/BigQuery|BigQuery]] and [[Storage/Cloud-Storage|Cloud Storage]].
- Define naming conventions and ownership for DAGs.
- Set up connections and secrets in [[Security/Secret-Manager|Secret Manager]].
- Configure alerting for task failures and SLA misses.
- Document retry and backfill strategy per pipeline.
- Enforce idempotent tasks — partition overwrites, MERGE, or WRITE_TRUNCATE.
