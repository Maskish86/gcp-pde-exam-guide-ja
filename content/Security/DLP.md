# DLP

Cloud Data Loss Prevention (DLP) helps discover, classify, and de-identify sensitive data. It detects patterns (PII, financial data, secrets) and can mask, tokenize, or redact fields to support privacy compliance.

## Use Cases
- Identify sensitive fields in raw data before analytics.
- De-identify data for broader sharing and consumer analyses.
- Enforce privacy requirements without losing utility.
- Monitor data lakes for policy violations.

## Mental Model
- DLP inspects data and returns findings based on infoTypes.
- De-identification transforms are applied before data is shared.
- Encryption at rest (CMEK) is not a substitute for de-identification.

## Core Concepts
- InfoType: detector for a sensitive pattern (email, SSN, credit card).
- Inspection: scan data for findings.
- De-identification: transform sensitive data (mask, tokenize, redact).
- Job: scheduled or triggered scan on storage or BigQuery.

## Common De-identification Options
- Masking (partial or full redaction).
- Tokenization (format-preserving tokens).
- Bucketing or generalization for quasi-identifiers.

## Typical Privacy Pipeline
- Read restricted data from [[Cloud-Storage|Cloud Storage]].
- Use [[Processing/Dataflow|Dataflow]] with DLP to mask or tokenize sensitive fields.
- Write de-identified data to [[Storage/BigQuery|BigQuery]] for consumer analytics.

## Integrations
- [[Cloud-Storage|Cloud Storage]]: scan and de-identify files in restricted buckets.
- [[Processing/Dataflow|Dataflow]]: streaming/batch pipelines with DLP transforms.
- [[Storage/BigQuery|BigQuery]]: inspect tables and write de-identified datasets.

## Common Pitfalls
- Treating encryption as privacy; DLP is for de-identification.
- Over-masking and losing analytic utility.
- Using generic detectors without tuning for false positives.

## Quick Checklist
- Define required infoTypes and inspection rules.
- Choose de-identification method per field.
- Process data in [[Processing/Dataflow|Dataflow]] and write to curated datasets.
- Validate utility and privacy with sample outputs.
- Monitor DLP findings and adjust rules.
