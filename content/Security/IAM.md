# IAM

Identity and Access Management (IAM) controls who can do what on which GCP resources. It is the core of least-privilege access for data platforms.

## Use Cases
- Grant access to datasets, buckets, and pipelines.
- Separate admin duties from data access.
- Control service account permissions for jobs and workflows.

## Mental Model
- Policies bind principals to roles on resources.
- Roles are collections of permissions.
- Deny by default; grant only what is needed.

## Core Concepts
- Principal: user, group, service account, or domain.
- Role: predefined or custom set of permissions.
- Policy: bindings of principals to roles on a resource.
- Resource hierarchy: org -> folder -> project -> resource.

## Service Accounts
- Use for workloads and pipelines, not humans.
- Rotate keys and avoid long-lived keys when possible.
- Grant minimal roles per workload.

**Credential hierarchy (prefer top over bottom):**
1. **Attached service account** (VM, Cloud Run, Dataflow job) — no key file created; credentials are short-lived, auto-rotating, fetched via the metadata server. This is the recommended approach.
2. **Workload Identity Federation** — for workloads outside GCP; exchanges external identity tokens for short-lived GCP credentials.
3. **[[Secret-Manager\|Secret Manager]]** — only if a key file is truly unavoidable; better than hardcoding, but still a long-lived credential that must be rotated manually.

> Storing a service account key in Secret Manager is an improvement over hardcoding it, but the correct answer is usually to avoid creating a key file at all by attaching the service account directly to the resource.

## Security And Governance
- Use least privilege and resource-level bindings.
- Separate admin roles from data access roles.
- Use audit logs to monitor access and policy changes.
- Use VPC Service Controls to create service perimeters that block cross-project data access.
- Cloud Logging: `roles/logging.viewer` cannot read private Data Access logs; use `roles/logging.privateLogViewer` for audit visibility.

## Design Tips
- Prefer group-based access for humans.
- Use conditional IAM for time or resource constraints.
- Avoid over-broad roles like Owner/Editor in production.

## Integrations
- [[Cloud-SQL|Cloud SQL]]: Auth proxy/connectors use `roles/cloudsql.client` for secure access.
- [[Cloud-Storage|Cloud Storage]] / [[Storage/BigQuery|BigQuery]]: dataset and bucket access control.
- [[Processing/Dataflow|Dataflow]] / [[Processing/Dataproc|Dataproc]]: service accounts for job execution.

## Common Pitfalls
- Granting broad roles to users instead of service accounts.
- Forgetting to remove temporary access.
- Relying only on network allowlists without IAM controls.
- Assuming IAM conditions alone prevent data exfiltration across projects.
## Quick Checklist
- Define the resource boundary.
- Choose least-privilege roles.
- Bind groups/service accounts, not individuals.
- Enable audit log monitoring.
