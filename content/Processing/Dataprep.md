# Dataprep

Cloud Dataprep (powered by Trifacta) is a managed, visual data preparation tool for profiling, cleaning, and transforming data in [[Cloud-Storage|Cloud Storage]] or [[Storage/BigQuery|BigQuery]]. It generates [[Processing/Dataflow|Dataflow]] jobs under the hood for execution — there is no separate Dataprep execution engine.

## Mental Model
- Dataprep is a **UI on top of Dataflow** — it abstracts pipeline authoring into a visual recipe editor.
- It targets **analysts and data stewards**, not engineers.
- Execution is **always batch** — Dataflow for GCS data, BigQuery for BQ data.
- Not a replacement for Dataflow or Data Fusion in production pipelines.

## Exam Domain
- Designing data processing systems
- Building and operationalizing data processing systems
- Ensuring solution quality

## Core Concepts

| Concept     | Description                                               |
| ----------- | --------------------------------------------------------- |
| Flow        | Dataprep project: connects datasets, recipes, and outputs |
| Recipe      | Ordered sequence of cleaning/transformation steps         |
| Transformer | Visual editor for building recipes                        |
| Dataset     | Source data from GCS or BigQuery                          |
| Job         | Execution of a flow (runs as a Dataflow job)              |

## Constraints
- Latency: interactive/adhoc prep; not low-latency streaming
- Scale: moderate; can offload execution to [[Processing/Dataflow|Dataflow]] or [[Storage/BigQuery|BigQuery]]
- Consistency: batch, deterministic transformations
- Cost: pay for the underlying execution engine; avoid for trivial SQL-only transforms
- Operational overhead: low (managed visual tool)
- Batch vs streaming: **batch only**

| Service          | Best When                                                    | Fails When                                                   |
| ---------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **Dataprep**     | Visual cleaning, analyst-driven prep, minimal coding         | Streaming pipelines, complex multi-stage engineering workflows |
| **Dataflow**     | Large-scale batch/streaming with custom transforms           | Business users need a GUI; quick one-off cleaning            |
| **Data Fusion**  | Reusable ETL with many connectors and governance             | Ad-hoc prep; overhead too high for small jobs                |
| **BigQuery SQL** | SQL-only transforms on BigQuery data                         | File-based prep or non-SQL cleaning required                 |

## Correct Choice
- Pick **Dataprep** when the constraint is **low engineering effort + visual data prep** on files or BigQuery tables and batch execution is acceptable.
- Pick **Dataflow** when custom logic or streaming is required.
- Pick **Data Fusion** when you need production ETL pipelines with many connectors and governance.
- Pick **BigQuery SQL** when transformations are pure SQL in BigQuery.

## Why Not the Others
- **Dataflow:** overkill for analyst-driven prep; higher dev/ops effort.
- **Data Fusion:** heavier ops/cost for simple or one-off cleaning.
- **BigQuery SQL:** cannot handle non-SQL prep or file-centric workflows well.

## Common Exam Traps
- "Visual, self-service data cleaning by analysts" → **Dataprep**, not Dataflow or Data Fusion.
- "Dataprep execution engine" doesn't exist → it **generates and runs Dataflow jobs**; cost = Dataflow worker hours + Dataprep service fee.
- Choosing Dataprep for **streaming** → it is **batch only**; use Dataflow or Pub/Sub for real-time pipelines.
- Choosing Dataprep for **complex, multi-stage production ETL** → prefer **Data Fusion** (connectors + governance) or **Dataflow** (custom transforms).
- Choosing Dataprep when the source is **not GCS or BigQuery** → Dataprep only reads from GCS and BigQuery; other sources require Dataflow or Data Fusion.
- "Minimize cost for SQL-only BigQuery transforms" → **BigQuery SQL**, not Dataprep — using Dataprep adds unnecessary Dataflow worker overhead for pure SQL work.
