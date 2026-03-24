# Data Fusion

Cloud Data Fusion is GCP's managed, visual ETL/ELT pipeline service built on open-source CDAP. It provides a drag-and-drop Studio with 150+ pre-built connectors, runs pipelines on [[Processing/Dataproc|Dataproc]] (Spark) under the hood, and is designed for **production, connector-rich, governed ETL** — not ad-hoc prep or custom streaming code.

## Mental Model
- Data Fusion = **visual pipeline builder on top of Dataproc (Spark)** — you design; Spark executes.
- Targets **data engineers and integration teams**, not ad-hoc analysts.
- Primary strength is **connector breadth** (databases, SaaS, files, messaging) and **pipeline reuse**.
- Higher ops/cost than Dataprep; lower engineering effort than Dataflow.
- Not the first choice for sub-second streaming or simple SQL-only transforms.

## Exam Domain
- Designing data processing systems
- Building and operationalizing data processing systems
- Ensuring solution quality

## Core Concepts

| Concept       | Description                                                       |
| ------------- | ----------------------------------------------------------------- |
| Studio        | Visual drag-and-drop pipeline builder                             |
| Pipeline      | ETL/ELT workflow composed of sources, transforms, and sinks       |
| Plugin        | Connector unit: source, sink, transform, or action                |
| Hub           | Marketplace for pre-built plugins, pipelines, and datasets        |
| Wrangler      | Interactive data cleaning tool embedded in Data Fusion            |
| Namespace     | Logical isolation boundary for pipelines and resources            |
| Replication   | CDC-based replication feature (uses Datastream under the hood)    |

## Editions

| Edition         | Key Additions                                                       |
| --------------- | ------------------------------------------------------------------- |
| **Developer**   | For development and testing only; not for production                |
| **Basic**       | Production workloads, standard connectors                           |
| **Enterprise**  | Data lineage, metadata management, private VPC, row-level security  |

## Constraints
- Latency: batch and micro-batch; not the first choice for sub-second streaming
- Scale: medium to large; depends on underlying execution engine
- Consistency: deterministic batch pipelines
- Cost: higher than SQL-only or simple batch; justified by connector breadth and governance
- Operational overhead: moderate (managed service but pipeline lifecycle and plugins)
- Batch vs streaming: mainly batch; streaming supported but not primary exam default

| Service          | Best When                                                        | Fails When                                               |
| ---------------- | ---------------------------------------------------------------- | -------------------------------------------------------- |
| **Data Fusion**  | Managed ETL with many connectors, reusable pipelines, governance | Very low-latency streaming or simple SQL transforms      |
| **Dataflow**     | High-scale batch/streaming with custom code                      | Business users need GUI/low-code; connector-rich ETL     |
| **Dataprep**     | Visual, ad-hoc data cleaning                                     | Production ETL with multiple sources and governance      |
| **BigQuery SQL** | Pure SQL transformations in BigQuery                             | Multi-source ETL or complex non-SQL transforms           |

## Correct Choice
- Pick **Data Fusion** when the constraints emphasize **connector-rich ETL**, **reusable pipelines**, and **managed governance** with moderate ops effort.
- Pick **Dataflow** for custom code, large-scale batch/streaming.
- Pick **Dataprep** for analyst-driven, visual, ad-hoc prep.
- Pick **BigQuery SQL** when transformations are SQL-only in BigQuery.

## Why Not the Others
- **Dataflow:** higher engineering effort; not ideal for GUI-driven, connector-heavy ETL.
- **Dataprep:** designed for ad-hoc prep, not production ETL with many sources.
- **BigQuery SQL:** limited to BQ data and SQL transformations.

## Common Exam Traps
- Choosing **Dataflow** for a "low-code ETL with many connectors" requirement → points to **Data Fusion**.
- Confusing **Data Fusion** with **Dataprep** — both are visual/low-code, but Data Fusion is for production multi-source ETL; Dataprep is for analyst-driven, ad-hoc cleaning.
- Choosing Data Fusion for **sub-second streaming** → it is batch/micro-batch first; use **Dataflow** for real-time.
- Assuming Data Fusion is cost-efficient for small or frequent jobs → Dataproc cluster startup overhead makes it expensive for trivial workloads.
