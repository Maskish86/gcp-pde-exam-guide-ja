# Apache

## Apacheエコシステム

[Apache Software Foundation (ASF)](https://apache.org) は非営利団体で、350+ のオープンソースプロジェクトを統括しており、クラウドネイティブなサービスが主流になる前に使われていたビッグデータ基盤ツールの多くの源流でもある。2000年代〜2010年代にかけて、オープンソースコミュニティが独立に実装した3本の影響力の大きいGoogle研究論文を起点として、ビッグデータ基盤の中核になった。

- **Google File System (2003)** → **HDFS** の着想元
- **MapReduce (2004)** → **Hadoop MapReduce** の着想元
- **Bigtable (2006)** → **Apache HBase** の着想元

これらを起点に、レイヤードなエコシステム全体が発展した。

| レイヤー        | 主要ツール                        | 目的                                         |
| --------------- | -------------------------------- | -------------------------------------------- |
| **コンピュート** | Hadoop, Spark, Flink             | 大規模データの処理                            |
| **ストレージ**   | HDFS, HBase                      | 大規模データセットの保存と取得                 |
| **SQL**         | Hive, Impala                     | SQLでデータをクエリする                       |
| **メッセージング** | Kafka, Pulsar                    | システム間のイベントストリーミング             |
| **オーケストレーション** | Airflow, Oozie                 | ジョブのスケジューリングと調整                |
| **ガバナンス**   | Atlas, ZooKeeper                 | メタデータ管理とクラスタ協調                  |

**コアのトレードオフ:** Apacheツールは強力で柔軟だが、運用負荷が大きい。クラスタのプロビジョニング、設定管理、スケーリング、アップグレード維持を自分で担う必要がある。GCPのマネージドサービスは、同じアーキテクチャパターンを保ちながら、この運用負荷を置き換える。

> **コア原則:** Apache = 自分で運用するインフラ。GCP = そのレイヤーを置き換えるマネージドサービス。

## 分散処理 / コンピュート

- [[Processing/Dataproc\|Dataproc]] は、マネージドクラスタ上で Hadoop と Spark をネイティブに動かす。既存ジョブを持ち込み、プロビジョニングとパッチ適用はGCPが担う。
- [[Processing/Dataflow\|Dataflow]] はさらに踏み込み、Apache Beamモデルでフルサーバレスかつ自動スケールのパイプラインを提供する。管理すべきクラスタは存在しない。
- [[Dataproc]] はSpark/HadoopのAPIサーフェスを維持する（コード変更最小でリフト＆シフト向き）。一方、[[Dataflow]] はランナーを完全に抽象化する（新規ストリーミングパイプラインではこちらが第一候補）。
- FlinkはDataflowに対応づけられる。どちらもステートフルでイベント時刻（event-time）前提のストリーム処理に向く。

| Apache        | 役割                         | GCP相当                                                                               | 補足                                           |
| ------------- | --------------------------- | ------------------------------------------------------------------------------------- | ---------------------------------------------- |
| Apache Hadoop | バッチ処理エコシステム         | [[Processing/Dataproc\|Dataproc]]                                                     | Hadoopジョブのリフト＆シフト                   |
| Apache Spark  | バッチ + ストリーミング計算     | [[Processing/Dataproc\|Dataproc]] / [[Processing/Dataflow\|Dataflow]]                 | Dataproc = ネイティブSpark、Dataflow = Beamモデル |
| Apache Flink  | リアルタイムのストリーム処理    | [[Processing/Dataflow\|Dataflow]]                                                     | 概念的な対応                                   |

## ストレージ

- Hadoop近代化における本質的なシフト：**HDFSはストレージをクラスタノードに結合する**。データ保持のためにクラスタを生かし続ける必要がある。一方、[[Storage/Cloud-Storage\|Cloud Storage]] は完全に分離するため、クラスタは起動→実行→停止してもデータを失わない。
- [[OperationalDBs/Bigtable\|Bigtable]] はHBaseの直接的な相当先である。HBaseはそもそもGoogle社内のBigtable論文に着想を得ており、同じワイドカラム + 行キー（row-key）モデルを共有し、大規模かつ低レイテンシ読み取りに最適化されている。
- Cassandraの対応先はアクセスパターン次第：ワイドカラムで低レイテンシ → **Bigtable**。マルチリージョンで強整合 → [[OperationalDBs/Spanner\|Spanner]]。

| Apache                          | 役割                        | GCP相当                                                                                | 補足                      |
| ------------------------------- | -------------------------- | -------------------------------------------------------------------------------------- | ------------------------- |
| Hadoop Distributed File System  | 分散ファイルストレージ       | [[Storage/Cloud-Storage\|Cloud Storage]]                                               | HDFSの置き換え             |
| Apache HBase                    | 低レイテンシ大規模DB         | [[OperationalDBs/Bigtable\|Bigtable]]                                                  | 概念的に1:1                |
| Apache Cassandra                | 高可用なNoSQL               | [[OperationalDBs/Bigtable\|Bigtable]] / [[OperationalDBs/Spanner\|Spanner]]            | アクセスパターン次第        |

## Data Warehouse / SQL

- HiveはMapReduceまたはTez経由でHDFS上にSQLを実行する。遅い、クラスタ依存、schema-on-read。[[Storage/BigQuery\|BigQuery]] はスタック全体を置き換える（サーバレス、カラムナ（Dremel engine）、クラスタ不要）。
- Impalaはインメモリ実行でHiveを高速化するために作られた。BigQueryは両方を包含し、インフラ運用オーバーヘッドなしでより高速である。
- 以前は2つのツールが必要だった領域をBigQueryが統合するため、どちらも同じGCPサービスに対応づけられる。

| Apache        | 役割                    | GCP相当                       | 補足                         |
| ------------- | ---------------------- | ------------------------------ | --------------------------- |
| Apache Hive   | HDFS上のデータウェアハウス | [[Storage/BigQuery\|BigQuery]] | サーバレス、クラスタ管理なし |
| Apache Impala | 高速なSQLクエリ         | [[Storage/BigQuery\|BigQuery]] | フルマネージド               |

## メッセージング / ストリーミング

- Kafkaはブローカー、パーティション、レプリケーション係数、コンシューマグループのオフセット管理が必要になる。[[Ingestion/PubSub\|Pub/Sub]] はそれらを不要にする（サーバレス、デフォルトでグローバル複製、設定なしでスケール）。
- トレードオフ：Kafkaはよりリッチなセマンティクス（log compaction、ブローカーでの exactly-once、保持期間の設定など）を提供する。Pub/Subはシンプルさとマネージドの信頼性を優先する。
- PulsarはKafka風のメッセージングに地理冗長（geo-replication）を組み込む。Pub/Subはより単純な抽象で同種のユースケースをカバーする。

| Apache        | 役割                   | GCP相当                      | 補足                    |
| ------------- | --------------------- | ----------------------------- | ---------------------- |
| Apache Kafka  | イベントストリーミング   | [[Ingestion/PubSub\|Pub/Sub]] | サーバレス代替           |
| Apache Pulsar | メッセージング + ストリーミング | [[Ingestion/PubSub\|Pub/Sub]] | 近い抽象                |

## オーケストレーション / ワークフロー

- [[Orchestration/Cloud-Composer\|Cloud Composer]] はマネージドのApache Airflowである。DAG、operator、hook、UIは同じで、基盤のGKEクラスタ、データベース、スケジューラはGCPが管理する。
- 対応はほぼ1:1であり、既存Airflowパイプラインの移行先として、最小限のコード変更で自然に選べる。
- OozieはHDFSとMapReduceに密結合である。Composerは、より広いコネクタエコシステムと、Pythonベースの現代的なパイプライン記述で置き換える。

| Apache         | 役割                      | GCP相当                                            | 補足               |
| -------------- | ------------------------ | --------------------------------------------------- | ------------------ |
| Apache Airflow | DAGベースのパイプライン     | [[Orchestration/Cloud-Composer\|Cloud Composer]]    | ほぼ1:1             |
| Apache Oozie   | Hadoopジョブのオーケストレーション | [[Orchestration/Cloud-Composer\|Cloud Composer]]    | 現代的な置き換え     |

## データ統合 / ETL

- NiFiと [[Processing/Data-Fusion\|Data Fusion]] はどちらも可視化されたフローベースの統合ツールである。Data Fusion（CDAP上）はETL志向で、150+ の事前構築コネクタにより本番パイプラインに向く。一方でNiFiはリアルタイムルーティングと詳細なプロビナンス（来歴）を重視する。
- SqoopはHDFSとリレーショナルデータベース間のバッチ転送ツールだった。GCPでは2つの経路に分かれる：継続的なCDCレプリケーションは [[Ingestion/Datastream\|Datastream]]、カスタムのバッチ転送パイプラインは [[Processing/Dataflow\|Dataflow]]。

| Apache       | 役割                      | GCP相当                                                                                  | 補足          |
| ------------ | ------------------------ | ---------------------------------------------------------------------------------------- | ------------- |
| Apache NiFi  | データ取り込みパイプライン  | [[Processing/Data-Fusion\|Data Fusion]]                                                  | GUIベースETL  |
| Apache Sqoop | データ転送                 | [[Ingestion/Datastream\|Datastream]] / [[Processing/Dataflow\|Dataflow]]                 | CDC vs バッチ |

## 機械学習

- MahoutはHadoop MapReduce上でMLを実行していたが、現在は概ね時代遅れである。Vertex AIが現代的な置き換えで、マネージドAutoML、カスタム学習、特徴量ストア、モデル提供をクラスタ管理なしで提供する。
- Spark MLlibはSpark上で分散MLをネイティブに実行する。GCPでは、既存MLlibコードを維持するなら [[Processing/Dataproc\|Dataproc]]、学習から提供までをフルマネージドにするなら Vertex AI へ移行する。

| Apache             | 役割          | GCP相当                                       | 補足               |
| ------------------ | ------------ | --------------------------------------------- | ------------------ |
| Apache Mahout      | Hadoop上のML | Vertex AI                                     | 現代的なMLプラットフォーム |
| Apache Spark MLlib | ML pipelines | Vertex AI / [[Processing/Dataproc\|Dataproc]] | 用途次第            |

## 協調 / メタデータ

- ZooKeeperは、Kafka、HBase、Hadoopが依存する分散協調（リーダー選出、ロック、設定）を担う。GCPではこれらの関心事が **完全に抽象化** される。Pub/Sub、Bigtable、Dataprocは内部で協調を管理するため、アーキテクチャからZooKeeperが消える。
- Apache AtlasはHadoopエコシステム向けにメタデータ、リネージ、ガバナンスを提供する。GCPでは [[Governance/Data-Catalog\|Data Catalog]] が相当し、BigQuery、Cloud Storage、Pub/Sub、Bigtableへネイティブ統合されている。

| Apache           | 役割                  | GCP相当                                           | 補足            |
| ---------------- | -------------------- | ------------------------------------------------- | --------------- |
| Apache ZooKeeper | クラスタ協調           | GCPサービス内部で管理                              | 抽象化される     |
| Apache Atlas     | メタデータ / カタログ   | [[Governance/Data-Catalog\|Data Catalog]]         | ガバナンス       |

## 使わない場面

**Pub/Sub:**
- すべてのメッセージで厳密な順序が必要 → ordering keys（並列性が下がる）またはKafka
- リプレイ可能なイベントログ（保持期間を自由に設定） → Kafka、または [[Storage/Cloud-Storage\|Cloud Storage]] に着地させる
- サブミリ秒レイテンシ → Pub/Subは~100ms程度増えるため、代わりに [[OperationalDBs/Memorystore\|Memorystore]] を使う

**Dataflow:**
- 既存Spark/Hadoopコードがあり、リファクタ予算が小さい → [[Processing/Dataproc\|Dataproc]]
- ネイティブSpark API（MLlib、GraphX）が必要 → Dataproc
- 長時間稼働の固定クラスタでコスト重視 → Dataprocの方が安い

**Dataproc:**
- フルサーバレス要件 → Dataprocはクラスタライフサイクル管理が必要 → Dataflow または [[Storage/BigQuery\|BigQuery]]
- 既存Sparkコードなしで新規パイプライン設計 → 一度だけ移行コストを払い、Dataflowを使う

**BigQuery:**
- 低レイテンシOLTP読み取り（sub-10ms） → [[OperationalDBs/Spanner\|Spanner]] または [[OperationalDBs/Cloud-SQL\|Cloud SQL]]
- 小さな書き込みが高頻度 → BigQueryはバルクロード最適化のため、OLTP DB + 定期バッチロードを使う

**Cloud Composer:**
- シンプルな直列ワークフロー → Cloud Workflows（軽量でAirflowオーバーヘッドなし）
- 軽量なイベント駆動タスク → Cloud Functions / Cloud Run

**Data Fusion:**
- リアルタイムストリーミング → [[Processing/Dataflow\|Dataflow]]
- 軽微または一回限りの変換 → [[Processing/Dataprep\|Dataprep]] または BigQuery SQL

## セキュリティとガバナンスのマッピング

| 関心事                            | GCPサービス                                         |
| -------------------------------- | --------------------------------------------------- |
| データ分類 / PII検出                | [[Security/DLP\|DLP]]                            |
| メタデータ管理 / リネージ            | [[Governance/Data-Catalog\|Data Catalog]]           |
| 暗号化（顧客管理）                  | [[Security/Cloud-KMS\|Cloud KMS]] / CMEK            |
| アクセス制御                        | [[Security/IAM\|IAM]] + [[Security/VPC-Service-Controls\|VPC-SC]] |
| 統合データガバナンス                 | [[Governance/Dataplex\|Dataplex]]                   |
| 監査ログ                            | Cloud Logging + Cloud Audit Logs                    |

## コストと性能のトレードオフ

**Dataproc vs Dataflow:**

| 要素              | Dataproc                              | Dataflow                            |
| ----------------- | ------------------------------------- | ----------------------------------- |
| コストモデル       | クラスタ時間（稼働中は課金）            | 実際に使った vCPU-hour 課金          |
| 運用オーバーヘッド | 高い（クラスタライフサイクル管理）       | 低い（フルサーバレス）               |
| 最適              | 長時間稼働クラスタ、SparkネイティブAPI   | 短命ジョブ、予測不能な負荷            |
| 選ぶ場面          | 既存クラスタ/Sparkコードの再利用         | 新規パイプライン、オートスケールが必要 |

**BigQuery pricing:**
- **On-demand**: スキャンTBあたり課金 — アドホックで予測不能なワークロードに向く
- **Flat-rate / capacity**: 固定スロット予約 — 予測可能で高ボリュームの本番ワークロードに向く
- パーティションフィルタはスキャンコストを下げ、クラスタリングは追加コストなしでスキャンバイトをさらに削減する

## 判断ルール

**最小変更 vs 再設計:**

| 要件                                | 回答                                                |
| ---------------------------------- | --------------------------------------------------- |
| Preserve existing Spark/Hadoop code | [[Processing/Dataproc\|Dataproc]]                  |
| Rewrite for serverless architecture | [[Processing/Dataflow\|Dataflow]]                  |
| Existing Airflow DAGs              | [[Orchestration/Cloud-Composer\|Cloud Composer]]    |
| Simple linear workflow (no DAG)    | Cloud Workflows                                     |

**取り込みパターン:**

| パターン                         | GCPサービス                                                           |
| ------------------------------- | -------------------------------------------------------------------- |
| リアルタイムDB同期（CDC）         | [[Ingestion/Datastream\|Datastream]]                                 |
| バッチのファイル取り込み          | [[Processing/Dataflow\|Dataflow]] / Storage Transfer Service         |
| アプリからのイベントストリーミング | [[Ingestion/PubSub\|Pub/Sub]]                                        |
| 一度きりの大量移行                | Storage Transfer Service / `gsutil`                                  |
| ホットデータ（高頻度・低レイテンシ） | [[OperationalDBs/Bigtable\|Bigtable]] / [[OperationalDBs/Memorystore\|Memorystore]] |
| コールドデータ（アーカイブ）        | [[Storage/Cloud-Storage\|Cloud Storage]]（Coldline / Archive）        |

## 主要パターン

### Hadoopのリフト＆シフト
```
Hadoop  → Dataproc
HDFS    → Cloud Storage
Airflow → Composer
```

### アーキテクチャの現代化
```
Kafka       → Pub/Sub
Spark/Flink → Dataflow
Hive        → BigQuery
```

### GUI / ローコードのパイプライン
```
NiFi  → Data Fusion
Sqoop → Datastream
```

## 意思決定ルール

| 見かけたら…          | GCPサービス                                           |
| -------------------- | ---------------------------------------------------- |
| Hadoopジョブ           | [[Processing/Dataproc\|Dataproc]]                    |
| サーバレス分析          | [[Storage/BigQuery\|BigQuery]]                       |
| ストリーミングパイプライン | [[Processing/Dataflow\|Dataflow]]                    |
| イベント取り込み         | [[Ingestion/PubSub\|Pub/Sub]]                        |
| Airflow DAG            | [[Orchestration/Cloud-Composer\|Cloud Composer]]     |
| GUI ETL               | [[Processing/Data-Fusion\|Data Fusion]]              |
| CDCレプリケーション      | [[Ingestion/Datastream\|Datastream]]                 |
