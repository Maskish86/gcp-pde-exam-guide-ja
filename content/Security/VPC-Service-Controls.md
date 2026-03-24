# VPC Service Controls

VPC Service Controls (VPC SC) create a service perimeter around Google-managed services to reduce data exfiltration risk. They restrict access based on projects and services, even if IAM permissions would otherwise allow it.

## Why You Use It
- Prevent cross-project access to sensitive services (e.g., [[Ingestion/PubSub|Pub/Sub]], [[Storage/BigQuery|BigQuery]], [[Cloud-Storage|Cloud-Storage]]).
- Block access from outside the defined perimeter, including accidental or malicious use of credentials.
- Add a perimeter-based control to complement IAM for sensitive data.

## Key Points
- VPC SC applies at the project/service (Google API) level, not VPC network level.
- It protects Google-managed services, not VM-to-VM traffic.
- IAM alone is permission-based; VPC SC adds a boundary that can still deny access.
- Projects outside the perimeter cannot access protected services unless explicitly added.
- VPC SC is access control (perimeter) for Google APIs; it does **not** provide network reachability. For internal‑IP only workers (e.g., Dataflow) that must reach GCS/BQ without public internet, use **Private Google Access** (connectivity). 

## Common Pitfalls
- Assuming VPC firewall rules can restrict `pubsub.googleapis.com` access.
- Relying solely on IAM for perimeter-level isolation.
- Forgetting to add new projects to the perimeter when needed.
