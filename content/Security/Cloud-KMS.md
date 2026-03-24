# Cloud KMS

Cloud Key Management Service manages cryptographic keys for encrypting data in GCP and your applications. It provides centralized key lifecycle control, IAM-based access, and audit logging for all encryption operations.

## Use Cases
- Enable CMEK for services like [[Storage/BigQuery|BigQuery]] and [[Cloud-Storage|Cloud-Storage]].
- Control key rotation, disablement, and destruction policies.
- Separate encryption duties from data access (security + compliance).
- Encrypt data in custom applications or pipelines.

## Mental Model
- KMS uses **envelope encryption**: services encrypt data with DEKs; KMS protects the DEKs with your key-encryption-key (KEK).
- You grant access to **use** keys, not to read data — these are separate concerns.
- Key versions are immutable; rotation creates a new version, old versions remain for decrypting existing data.
- Disabling a key blocks new encrypt/decrypt operations; destruction is irreversible — disable first.

## Core Concepts

| Concept | Description |
| --- | --- |
| **Key ring** | Regional logical container for keys |
| **Crypto key** | Named key with one or more versions |
| **Key version** | Actual cryptographic material (enabled / disabled / destroyed) |
| **Key purpose** | Symmetric encrypt/decrypt or asymmetric sign/verify |
| **Software key** | Default protection level; key material in Google's infrastructure |
| **HSM key** | Hardware-backed, tamper-resistant; required by some compliance frameworks |
| **EKM key** | Key material stays in your on-prem/external key manager; never enters GCP |

## CMEK (Customer-Managed Encryption Keys)

Attach a KMS key to a dataset, bucket, or job — the service protects its DEKs with your key.

- BigQuery tables can’t switch from GMEK to CMEK in place; create a new CMEK table and copy data.

**Setting a bucket default CMEK** (enforces CMEK on all future writes):
```bash
gcloud storage buckets update gs://my-bucket \
  --default-encryption-key=projects/.../cryptoKeys/my-key
```

**Migration gotcha — two independent controls:**

| Control | What It Covers |
| --- | --- |
| Explicit key during copy/rewrite | Re-encrypts existing objects at that moment |
| Bucket default CMEK | Enforces CMEK on all future writes automatically |

Without setting the bucket default, later uploads can silently land under Google-managed encryption (GMEK). Always set both.

**Compromised key response:**
1. Create a new key and set it as the bucket/dataset default.
2. Rewrite/copy existing data — rotation alone does not re-encrypt stored objects.
3. Disable, then destroy the compromised key after re-encryption is confirmed.

## HSM And Cloud EKM

**Cloud HSM:** Keys backed by tamper-resistant hardware; cryptographic operations occur inside the HSM, raw key material never exposed. Use when compliance requires hardware assurance.

**Cloud EKM:** Key material stays in your on-prem HSM or external key manager. GCP services use the EKM-backed KMS key as CMEK — encryption is Google-managed, but the key never enters Google Cloud.

> If the requirement is "key material must stay exclusively on-prem", the answer is Cloud EKM. Importing keys into Cloud KMS or Cloud HSM places material in GCP. Client-side encryption is not a CMEK control for services like BigQuery at-rest encryption.

## Rotation And Lifecycle
- Set a rotation period and next rotation time on each key.
- New data uses the newest primary version automatically.
- Old versions remain active for decrypting previously encrypted data.
- Destroy only after confirming all data encrypted with that version has been re-encrypted or is no longer needed.

## Access Control And IAM

| Role | Use For |
| --- | --- |
| `roles/cloudkms.cryptoKeyEncrypterDecrypter` | Pipeline service accounts (encrypt + decrypt) |
| `roles/cloudkms.cryptoKeyEncrypter` | Write-only pipelines (encrypt only) |
| `roles/cloudkms.admin` | Key management — use sparingly, separate from key users |

- Scope access by project and key ring, not globally.
- Separate key admins from key users — principle of least privilege.

## Auditing And Compliance
- All key operations (create, use, disable, destroy) are logged in Cloud Audit Logs.
- Track decrypt events and policy changes to satisfy compliance requirements.
- Align rotation periods and log retention with your compliance framework.

## Common Pitfalls
- Destroying key versions before all encrypted data is re-encrypted — destruction is irreversible and causes permanent data loss; disable first, confirm no active data references the version, then destroy.
- One key for everything — a compromised or misconfigured key affects all protected data; scope keys by data domain or sensitivity tier, and separate key rings by environment.
- Forgetting to grant `cryptoKeyEncrypterDecrypter` to service accounts — operations fail silently with permission denied; the resource (BigQuery, GCS) holds the data but the service account needs explicit IAM on the key itself.
- Key rotation does not re-encrypt existing data — rotation creates a new primary version for future operations only; existing objects remain encrypted under the old version; explicitly rewrite or copy objects to re-encrypt them.
- Setting per-operation CMEK without a bucket or dataset default — future writes can silently fall back to GMEK (Google-managed encryption); always set the default CMEK on the resource, not just per-operation.
- Assuming CMEK satisfies privacy or PII requirements — CMEK controls encryption at rest, not data content; use [[Security/DLP|DLP]] for de-identification and PII redaction separately.
- Mismatched regions between KMS key and protected resource — cross-region key use adds latency and may violate data residency policy; key ring region must match the resource region; global keys bypass residency guarantees.

## Integrations
- [[Cloud-Storage|Cloud Storage]]: bucket default CMEK for all new object writes.
- [[Storage/BigQuery|BigQuery]]: dataset/table CMEK and job-level CMEK for query workers.
- [[Processing/Dataflow|Dataflow]] / [[Processing/Dataproc|Dataproc]]: CMEK for worker disks and pipeline staging data.
- [[Security/IAM|IAM]]: least-privilege roles for key usage and key administration.

## Quick Checklist
-  Choose region and align with data residency requirements.
-  Decide: symmetric vs asymmetric, software vs HSM vs EKM.
-  Set rotation policy and document recovery/re-encryption steps.
-  Grant least-privilege IAM — separate key admins from key users.
-  Set bucket/dataset default CMEK, not just per-operation keys.
-  Enable Cloud Audit Log monitoring for key usage and policy changes.
