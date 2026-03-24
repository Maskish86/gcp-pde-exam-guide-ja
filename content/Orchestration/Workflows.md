# Workflows

Workflows is GCP's serverless orchestration service for coordinating APIs and cloud services. It executes YAML/JSON-defined steps with retries, conditionals, and parallel branches, without managing servers.

## Use Cases
- Orchestrate lightweight pipelines across managed services.
- Glue together APIs for ingestion, validation, and notifications.
- Run event-driven flows triggered by Pub/Sub or HTTP.
- Replace simple scripts with auditable, managed workflows.

## Mental Model
- Each step calls an API, runs logic, or waits; state is managed for you.
- Workflows is orchestration, not compute-heavy processing.
- Retries and error handling are defined at the step or workflow level.
- Long-running logic should be offloaded to services like [[Processing/Dataflow|Dataflow]].

## Core Concepts
- Workflow: the definition and deployment unit.
- Step: a single action (call, assign, switch, try/catch).
- Execution: a single run of a workflow.
- Connector: built-in integration for GCP APIs.

## Common Patterns
- Call Cloud Storage -> validate -> trigger BigQuery load.
- Fan out parallel API calls, then aggregate results.
- Retry transient failures with exponential backoff.
- Use subworkflows for shared logic.

## Offloading Complex Logic (Cloud Run)
- Use Cloud Run (or Cloud Functions) for complex business logic that exceeds the Workflows standard library.
- Small API payloads are a good fit; Workflows orchestrates, Cloud Run computes.
- Fully managed runtimes with fast startup and automatic scaling keep the solution simple.
- Choose Cloud Run for stateless HTTP compute; prefer GKE only if you need cluster-level control or custom runtimes.

## Integrations
- [[Cloud-Storage|Cloud Storage]]: trigger or validate file availability.
- [[Storage/BigQuery|BigQuery]]: start jobs and check results.
- [[Processing/Dataflow|Dataflow]]: launch batch/streaming jobs.
- [[Ingestion/PubSub|Pub/Sub]]: event triggers and notifications.
- [[Secret-Manager|Secret Manager]]: store API keys and tokens.

## Exam Decision: Workflows vs Composer vs Cloud Run
| Requirement signal | Choose | Reject | Why rejected (distractor elimination) |
|---|---|---|---|
| Lightweight API orchestration (retries/branches/parallel calls) | Workflows | Cloud Composer | Composer is heavier-weight; don’t pick it if you don’t need DAG scheduling/backfills. |
| Scheduling + backfills + many dependencies across jobs | Cloud Composer | Workflows | Workflows is great for glue, but it’s not the “Airflow-style” scheduler/backfill tool. |
| Custom compute/business logic behind HTTP | Cloud Run | Workflows | Workflows is not a general compute runtime; use it to orchestrate, not to run heavy logic. |

## Security And Access Control
- Use least-privilege [[Security/IAM|IAM]] roles for the workflow service account.
- Avoid embedding secrets in workflow definitions; use [[Secret-Manager|Secret Manager]].
- Prefer private connectivity when calling internal services.

## Operations And Reliability
- Use structured logging and include correlation IDs.
- Monitor executions and set alerts for failure rates.
- Keep workflows small and composable for maintainability.

## Common Pitfalls
- Putting heavy compute inside the workflow instead of a dedicated service.
- Ignoring idempotency, leading to duplicate actions on retry.
- Hardcoding endpoints instead of using configuration.
- Treating `kubectl scale` as autoscaling (use HPA or Cloud Run autoscale).

## Quick Checklist
- Define inputs and outputs clearly.
- Add retries and timeout handling for external calls.
- Store secrets in [[Secret-Manager|Secret Manager]].
- Add logging for key steps and errors.
- Monitor with [[Cloud-Monitoring|Cloud Monitoring]] and [[Cloud-Logging|Cloud Logging]].
