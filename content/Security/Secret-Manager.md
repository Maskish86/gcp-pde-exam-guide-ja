# Secret Manager

Secret Manager stores and manages sensitive configuration like API keys, passwords, and certificates. It provides versioned secrets, IAM-based access control, audit logs, and optional CMEK via [[Cloud-KMS|Cloud KMS]].

## Use Cases
- Store database credentials, API tokens, and service keys for pipelines.
- Centralize secrets for workflows ([[Processing/Dataflow|Dataflow]], [[Processing/Dataproc|Dataproc]], Cloud Run, Functions).
- Rotate secrets without code changes using versioned values.
- Reduce secret sprawl in env vars, files, or CI logs.

## Mental Model
- A secret is a named container; each value is a version.
- Apps reference a specific version (or "latest").
- Access is controlled per secret via IAM.
- Replication controls where secret data is stored.

## Core Concepts
- Secret: the resource name and metadata.
- Secret version: immutable payload; versions can be enabled/disabled/destroyed.
- Replication: automatic (multi-region) or user-managed (region-specific).
- Labels: metadata for ownership, cost, and lifecycle management.

## Access Control And IAM
Common roles:
- `roles/secretmanager.secretAccessor` to read secret versions.
- `roles/secretmanager.admin` to manage secrets (use sparingly).

Design tips:
- Grant access to service accounts, not users.
- Separate readers from admins.
- Use least privilege by secret and environment (dev/prod).

## Rotation And Lifecycle
- Create new versions for rotation; keep old versions briefly for rollback.
- Disable old versions before destroy to avoid accidental outage.
- Automate rotation via schedulers or CI.

## Encryption And CMEK
- Secrets are encrypted at rest by default.
- CMEK is supported via [[Cloud-KMS|Cloud KMS]] when you need key control and auditability.
- Align KMS key region with secret replication settings.

## Usage Patterns
- Read secrets at startup and cache them in memory.
- Avoid logging secret values or passing them as command-line args.
- Use version pinning for sensitive changes; use "latest" for auto-rotation.

## Integrations
- Cloud Run / Functions / GKE: inject secrets at runtime.
- [[Processing/Dataflow|Dataflow]] / [[Processing/Dataproc|Dataproc]]: read secrets for connectors and external systems.
- [[Security/IAM|IAM]]: governs access and separation of duties.

## Common Pitfalls
- Using Secret Manager for bulk data encryption (use [[Cloud-KMS|Cloud KMS]] instead).
- Leaving old versions enabled indefinitely.
- Over-broad IAM grants on secrets.
- Mixing secrets for different environments in one project.

## Quick Checklist
- Choose replication (auto vs user-managed).
- Set IAM for readers and admins (least privilege).
- Add labels for ownership and environment.
- Plan rotation cadence and automation.
- Enable audit log monitoring.
