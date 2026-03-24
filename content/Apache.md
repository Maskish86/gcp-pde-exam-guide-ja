## The Apache Ecosystem

The [Apache Software Foundation (ASF)](https://apache.org) is a non-profit that stewards 350+ open-source projects and is the source of most big data infrastructure tools used before cloud-native services became mainstream. It became the backbone of big data infrastructure through the 2000s and 2010s, largely driven by three influential Google research papers that the open-source community implemented independently:

- **Google File System (2003)** → inspired **HDFS**
- **MapReduce (2004)** → inspired **Hadoop MapReduce**
- **Bigtable (2006)** → inspired **Apache HBase**

From these roots, an entire layered ecosystem grew:

| Layer           | Key Tools                        | Purpose                                      |
| --------------- | -------------------------------- | -------------------------------------------- |
| **Compute**     | Hadoop, Spark, Flink             | Processing data at scale                     |
| **Storage**     | HDFS, HBase                      | Storing and retrieving large datasets        |
| **SQL**         | Hive, Impala                     | Querying data with SQL                       |
| **Messaging**   | Kafka, Pulsar                    | Event streaming between systems              |
| **Orchestration** | Airflow, Oozie                 | Scheduling and coordinating jobs             |
| **Governance**  | Atlas, ZooKeeper                 | Metadata management and cluster coordination |

**The core trade-off:** Apache tools are powerful and flexible but require significant operational effort — you provision clusters, manage configurations, handle scaling, and maintain upgrades yourself. GCP managed services replace this operational burden while preserving the same architectural patterns.

> **Core principle:** Apache = infrastructure you manage. GCP = managed service replacing that layer.

## Distributed Processing / Compute

- [[Processing/Dataproc\|Dataproc]] runs Hadoop and Spark natively on managed clusters — bring your existing jobs, GCP handles provisioning and patching.
- [[Processing/Dataflow\|Dataflow]] goes further: fully serverless, auto-scaling pipelines using the Apache Beam model — no cluster to manage at all.
- [[Dataproc]] preserves the Spark/Hadoop API surface (minimal code changes, good for lift-and-shift); [[Dataflow]] abstracts the runner entirely (preferred for new streaming pipelines).
- Flink maps to Dataflow — both are built for stateful, event-time stream processing.

| Apache        | Role                        | GCP Equivalent                                                                        | Notes                                          |
| ------------- | --------------------------- | ------------------------------------------------------------------------------------- | ---------------------------------------------- |
| Apache Hadoop | Batch processing ecosystem  | [[Processing/Dataproc\|Dataproc]]                                                     | Lift-and-shift Hadoop jobs                     |
| Apache Spark  | Batch + streaming compute   | [[Processing/Dataproc\|Dataproc]] / [[Processing/Dataflow\|Dataflow]]                 | Dataproc = native Spark, Dataflow = Beam model |
| Apache Flink  | Real-time stream processing | [[Processing/Dataflow\|Dataflow]]                                                     | Conceptual equivalent                          |

## Storage

- The foundational Hadoop modernisation shift: **HDFS couples storage to cluster nodes** — the cluster must stay alive to retain data; [[Storage/Cloud-Storage\|Cloud Storage]] decouples them entirely, so clusters can spin up, run, and shut down without data loss.
- [[OperationalDBs/Bigtable\|Bigtable]] is the direct HBase equivalent — HBase was actually inspired by Google's internal Bigtable paper, sharing the same wide-column, row-key model optimised for low-latency reads at scale.
- Cassandra's mapping depends on access pattern: wide-column low-latency → **Bigtable**; multi-region strong consistency → [[OperationalDBs/Spanner\|Spanner]].

| Apache                          | Role                       | GCP Equivalent                                                                         | Notes                     |
| ------------------------------- | -------------------------- | -------------------------------------------------------------------------------------- | ------------------------- |
| Hadoop Distributed File System  | Distributed file storage   | [[Storage/Cloud-Storage\|Cloud Storage]]                                               | Replaces HDFS             |
| Apache HBase                    | Low-latency large-scale DB | [[OperationalDBs/Bigtable\|Bigtable]]                                                  | 1:1 conceptual mapping    |
| Apache Cassandra                | Highly available NoSQL     | [[OperationalDBs/Bigtable\|Bigtable]] / [[OperationalDBs/Spanner\|Spanner]]            | Depends on access pattern |

## Data Warehouse / SQL

- Hive runs SQL on HDFS via MapReduce or Tez — slow, cluster-dependent, schema-on-read. [[Storage/BigQuery\|BigQuery]] replaces the entire stack: serverless, columnar (Dremel engine), no cluster needed.
- Impala was built to speed up Hive with in-memory execution; BigQuery subsumes both and is faster with zero infrastructure overhead.
- Both map to the same GCP service because BigQuery consolidates what previously required two separate tools.

| Apache        | Role                   | GCP Equivalent                 | Notes                       |
| ------------- | ---------------------- | ------------------------------ | --------------------------- |
| Apache Hive   | Data warehouse on HDFS | [[Storage/BigQuery\|BigQuery]] | Serverless, no cluster mgmt |
| Apache Impala | Fast SQL queries       | [[Storage/BigQuery\|BigQuery]] | Fully managed               |

## Messaging / Streaming

- Kafka requires managing brokers, partitions, replication factors, and consumer group offsets. [[Ingestion/PubSub\|Pub/Sub]] removes all of that — serverless, globally replicated by default, scales without configuration.
- The trade-off: Kafka offers richer semantics (log compaction, exactly-once at the broker, configurable retention); Pub/Sub favours simplicity and managed reliability.
- Pulsar adds built-in geo-replication on top of Kafka-like messaging; Pub/Sub covers the same use cases under a simpler abstraction.

| Apache        | Role                  | GCP Equivalent                | Notes                  |
| ------------- | --------------------- | ----------------------------- | ---------------------- |
| Apache Kafka  | Event streaming       | [[Ingestion/PubSub\|Pub/Sub]] | Serverless alternative |
| Apache Pulsar | Messaging + streaming | [[Ingestion/PubSub\|Pub/Sub]] | Similar abstraction    |

## Orchestration / Workflow

- [[Orchestration/Cloud-Composer\|Cloud Composer]] is managed Apache Airflow — same DAGs, operators, hooks, and UI; GCP handles the underlying GKE cluster, database, and scheduler.
- The mapping is almost 1:1, making Composer the natural migration target for existing Airflow pipelines with minimal code changes.
- Oozie is tightly coupled to HDFS and MapReduce; Composer replaces it with a broader connector ecosystem and modern Python-based pipeline authoring.

| Apache         | Role                     | GCP Equivalent                                      | Notes              |
| -------------- | ------------------------ | --------------------------------------------------- | ------------------ |
| Apache Airflow | DAG-based pipelines      | [[Orchestration/Cloud-Composer\|Cloud Composer]]    | Almost 1:1         |
| Apache Oozie   | Hadoop job orchestration | [[Orchestration/Cloud-Composer\|Cloud Composer]]    | Modern replacement |

## Data Integration / ETL

- NiFi and [[Processing/Data-Fusion\|Data Fusion]] are both visual, flow-based integration tools; Data Fusion (built on CDAP) is ETL-oriented with 150+ pre-built connectors suited for production pipelines, while NiFi emphasises real-time routing and fine-grained provenance.
- Sqoop was a batch transfer tool between HDFS and relational databases — on GCP this splits into two paths: [[Ingestion/Datastream\|Datastream]] for continuous CDC replication, [[Processing/Dataflow\|Dataflow]] for custom batch transfer pipelines.

| Apache       | Role                     | GCP Equivalent                                                                           | Notes         |
| ------------ | ------------------------ | ---------------------------------------------------------------------------------------- | ------------- |
| Apache NiFi  | Data ingestion pipelines | [[Processing/Data-Fusion\|Data Fusion]]                                                  | GUI-based ETL |
| Apache Sqoop | Data transfer            | [[Ingestion/Datastream\|Datastream]] / [[Processing/Dataflow\|Dataflow]]                 | CDC vs batch  |

## Machine Learning

- Mahout ran ML on Hadoop MapReduce and is now largely obsolete; Vertex AI is the modern replacement — managed AutoML, custom training, feature stores, and model serving with no cluster management.
- Spark MLlib runs distributed ML natively on Spark; on GCP, use [[Processing/Dataproc\|Dataproc]] to preserve existing MLlib code, or migrate to Vertex AI for a fully managed training and serving lifecycle.

| Apache             | Role         | GCP Equivalent                                | Notes              |
| ------------------ | ------------ | --------------------------------------------- | ------------------ |
| Apache Mahout      | ML on Hadoop | Vertex AI                                     | Modern ML platform |
| Apache Spark MLlib | ML pipelines | Vertex AI / [[Processing/Dataproc\|Dataproc]] | Depends on use     |

## Coordination / Metadata

- ZooKeeper handles distributed coordination (leader election, locks, config) that Kafka, HBase, and Hadoop depend on — on GCP these concerns are **abstracted away entirely**; Pub/Sub, Bigtable, and Dataproc manage their own coordination internally, so ZooKeeper disappears from the architecture.
- Apache Atlas provides metadata, lineage, and governance for the Hadoop ecosystem; [[Governance/Data-Catalog\|Data Catalog]] is its GCP equivalent, with native integration into BigQuery, Cloud Storage, Pub/Sub, and Bigtable.

| Apache           | Role                 | GCP Equivalent                                    | Notes           |
| ---------------- | -------------------- | ------------------------------------------------- | --------------- |
| Apache ZooKeeper | Cluster coordination | Managed internally in GCP services                | Abstracted away |
| Apache Atlas     | Metadata / catalog   | [[Governance/Data-Catalog\|Data Catalog]]         | Governance      |

## When NOT To Use

**Pub/Sub:**
- Strict ordering across all messages → use ordering keys (reduces parallelism) or Kafka
- Replayable event log with custom retention → Kafka or land to [[Storage/Cloud-Storage\|Cloud Storage]]
- Sub-millisecond latency → Pub/Sub adds ~100ms; use [[OperationalDBs/Memorystore\|Memorystore]] instead

**Dataflow:**
- Existing Spark/Hadoop code with minimal refactor budget → [[Processing/Dataproc\|Dataproc]]
- Native Spark APIs needed (MLlib, GraphX) → Dataproc
- Cost-sensitive workloads on long-running, fixed clusters → Dataproc is cheaper

**Dataproc:**
- Fully serverless requirement → Dataproc still requires cluster lifecycle management → Dataflow or [[Storage/BigQuery\|BigQuery]]
- New pipeline design with no existing Spark code → pay the migration cost once and use Dataflow

**BigQuery:**
- Low-latency OLTP reads (sub-10ms) → [[OperationalDBs/Spanner\|Spanner]] or [[OperationalDBs/Cloud-SQL\|Cloud SQL]]
- Frequent small writes → BigQuery is optimised for bulk loads; use an OLTP DB + periodic batch load

**Cloud Composer:**
- Simple linear workflows → Cloud Workflows (lighter, no Airflow overhead)
- Lightweight event-driven tasks → Cloud Functions / Cloud Run

**Data Fusion:**
- Real-time streaming → [[Processing/Dataflow\|Dataflow]]
- Trivial or one-off transforms → [[Processing/Dataprep\|Dataprep]] or BigQuery SQL

## Security And Governance Mapping

| Concern                          | GCP Service                                         |
| -------------------------------- | --------------------------------------------------- |
| Data classification / PII detection | [[Security/DLP\|DLP]]                            |
| Metadata management / lineage    | [[Governance/Data-Catalog\|Data Catalog]]           |
| Encryption (customer-managed)    | [[Security/Cloud-KMS\|Cloud KMS]] / CMEK            |
| Access control                   | [[Security/IAM\|IAM]] + [[Security/VPC-Service-Controls\|VPC-SC]] |
| Unified data governance          | [[Governance/Dataplex\|Dataplex]]                   |
| Audit logging                    | Cloud Logging + Cloud Audit Logs                    |

## Cost And Performance Trade-offs

**Dataproc vs Dataflow:**

| Factor            | Dataproc                              | Dataflow                            |
| ----------------- | ------------------------------------- | ----------------------------------- |
| Cost model        | Cluster time (pay while cluster runs) | Per vCPU-hour actually used         |
| Ops overhead      | Higher — cluster lifecycle to manage  | Lower — fully serverless            |
| Best for          | Long-running clusters, Spark-native APIs | Ephemeral jobs, unpredictable load |
| Choose when       | Reusing existing cluster / Spark code | New pipeline, auto-scaling needed   |

**BigQuery pricing:**
- **On-demand**: pay per TB scanned — good for ad-hoc, unpredictable workloads
- **Flat-rate / capacity**: fixed slot reservations — good for predictable, high-volume production workloads
- Partition filters reduce scan cost; clustering reduces bytes scanned further without extra cost

## Decision Rules

**Minimal change vs redesign:**

| Requirement                        | Answer                                              |
| ---------------------------------- | --------------------------------------------------- |
| Preserve existing Spark/Hadoop code | [[Processing/Dataproc\|Dataproc]]                  |
| Rewrite for serverless architecture | [[Processing/Dataflow\|Dataflow]]                  |
| Existing Airflow DAGs              | [[Orchestration/Cloud-Composer\|Cloud Composer]]    |
| Simple linear workflow (no DAG)    | Cloud Workflows                                     |

**Ingestion patterns:**

| Pattern                         | GCP Service                                                          |
| ------------------------------- | -------------------------------------------------------------------- |
| Real-time DB sync (CDC)         | [[Ingestion/Datastream\|Datastream]]                                 |
| Batch file ingestion            | [[Processing/Dataflow\|Dataflow]] / Storage Transfer Service         |
| Event streaming from apps       | [[Ingestion/PubSub\|Pub/Sub]]                                        |
| One-time bulk migration         | Storage Transfer Service / `gsutil`                                  |
| Hot data (frequent low-latency) | [[OperationalDBs/Bigtable\|Bigtable]] / [[OperationalDBs/Memorystore\|Memorystore]] |
| Cold data (archival)            | [[Storage/Cloud-Storage\|Cloud Storage]] (Coldline / Archive)        |

## Key Patterns

### Lift-and-Shift Hadoop
```
Hadoop  → Dataproc
HDFS    → Cloud Storage
Airflow → Composer
```

### Modernize Architecture
```
Kafka       → Pub/Sub
Spark/Flink → Dataflow
Hive        → BigQuery
```

### GUI / Low-Code Pipelines
```
NiFi  → Data Fusion
Sqoop → Datastream
```

## Quick Decision Heuristics

| If you see...        | GCP Service                                          |
| -------------------- | ---------------------------------------------------- |
| Hadoop jobs          | [[Processing/Dataproc\|Dataproc]]                    |
| Serverless analytics | [[Storage/BigQuery\|BigQuery]]                       |
| Streaming pipeline   | [[Processing/Dataflow\|Dataflow]]                    |
| Event ingestion      | [[Ingestion/PubSub\|Pub/Sub]]                        |
| Airflow DAGs         | [[Orchestration/Cloud-Composer\|Cloud Composer]]     |
| GUI ETL              | [[Processing/Data-Fusion\|Data Fusion]]              |
| CDC replication      | [[Ingestion/Datastream\|Datastream]]                 |
