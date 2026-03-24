# Cloud Composer

Cloud Composer は、ワークフローをオーケストレーションするためのGCPマネージド Apache Airflow サービスである。データシステム横断のタスクを調整するDAGをスケジュール/実行し、下位インフラはGoogleが管理する。

## ユースケース
- 複数のGCPサービスにまたがるELT/ETLパイプラインをスケジュールし、監視する。
- リトライとSLAを備えて、依存関係（ingest → transform → publish）を調整する。
- バックフィルやパラメータ実行が必要なバッチワークフローを動かす。
- 複数ステップのデータジョブに対する運用上の可視性を一元化する。

## メンタルモデル
- DAGがタスク依存を定義し、スケジューラが次に何を動かすかを決める。
- タスクは冪等であるべき（失敗時は自動的にリトライされる）。
- Composer は **オーケストレーションであり実行基盤ではない**：重い処理は [[Processing/Dataflow|Dataflow]]、[[Processing/Dataproc|Dataproc]]、[[Storage/BigQuery|BigQuery]] に置く。
- 環境の健全性は、DAG規模、スケジューリング負荷、ワーカー容量に依存する。

> 「複雑な依存関係を持つ複数ジョブ」→ Cloud Composer。  
> 「シンプルなAPI呼び出し連鎖/イベント駆動のつなぎ」→ [[Orchestration/Workflows|Workflows]]。

## コア概念

| 概念            | 説明                                                                                                                             |                  |
| --------------- | -------------------------------------------------------------------------------------------------------------------------------- | ---------------- |
| **Environment** | マネージドAirflow環境（scheduler、web UI、workers、metadata DB）                                                                  |                  |
| **DAG**         | タスク（ノード）と依存関係（エッジ）からなる有向非巡回グラフ。順序、スケジューリング/catchup、並列性を定義する（循環なし）       |                  |
| **Task**        | 作業単位（1つのoperatorインスタンス）                                                                                             |                  |
| **Operator**    | 事前定義のタスク種別（BigQuery、Dataflow、Bash、HTTPなど）                                                                        |                  |
| **Sensor**      | 外部条件を待ってから進む（ファイル、テーブル、メッセージなど）                                                                     |                  |
| **Executor**    | タスクをワーカーへどう分配し、どうスケールするかを制御する                                                                         |                  |
| **Connection**  | 外部システムの名前付き認証情報。シークレットは [[Security/Secret Manager                                               | Secret Manager]] に保存する |
| **REST API**    | 外部システムからDAG実行をトリガーし、タスクをクリアし、DAG/タスク状態を取得するAirflow API                                        |                  |

## 主要GCPオペレータ

| オペレータ | 用途 |
| --- | --- |
| `BigQueryInsertJobOperator` | BigQuery SQLジョブ（scheduled queries、DML、DDL）を投入する |
| `DataflowCreateJobOperator` | Dataflowのバッチ/ストリーミングジョブを起動する |
| `DataprocSubmitJobOperator` | DataprocクラスタへSpark/Hadoopジョブを投入する |
| `GCSObjectExistenceSensor` | Cloud Storageにファイルが到着するまで待つ |
| `BigQueryTableExistenceSensor` | テーブルまたはパーティションが生成されるまで待つ |
| `PubSubPullSensor` | Pub/Subメッセージを待つ |
| `BashOperator` | CLIコマンド（`bq`、`gcloud`、シェルスクリプト）を実行する |

## よくあるDAGパターン

**Ingest → stage → transform → publish:**
- ステップ間にsensorを置き、データ到着を待ってから次へ進む。
- 各タスクは冪等である必要がある（`WRITE_TRUNCATE`、`MERGE`、パーティション上書きを使う）。
- 変換が同一でない限り、テーブルごとにDAGを分ける。分岐が多い巨大DAGは避ける。

**Backfills:**
- `catchup=True` は、DAG有効化時に取りこぼした間隔を自動的にリプレイする。
- 手動: `airflow dags backfill -s <start_date> -e <end_date> <dag_id>`
- 各runが正しいパーティションを対象にできるよう、`{{ ds }}`（execution date）でタスクをパラメータ化する。

**SLA monitoring:**
- 個別タスクに `sla` を設定し、超過時にAirflowが `sla_miss_callback` を起動する。
- オンコールのエスカレーションには、メールアラートや [[Ingestion/PubSub|Pub/Sub]] 通知と組み合わせる。

**Scheduled BigQuery with retries:**
- `retries=3`、`retry_delay=timedelta(minutes=5)` を設定した `BigQueryInsertJobOperator` を使う。
- 冪等性要件に応じて `WRITE_APPEND` または `WRITE_TRUNCATE` を使う。
- リトライ枯渇後にアラートするため `email_on_failure=True` を設定する。
- 2時間SQLにリトライ/アラートが必要なら、Composer + `BigQueryInsertJobOperator` を優先する。Workflowsはスケジューリング/バックフィル用途としては軽すぎる。

## 試験の判断：Composer vs Workflows

| 要件シグナル                                                          | 選ぶ                                 | 却下                                 | 却下理由                                                                                       |
| --------------------------------------------------------------------- | -------------------------------------- | -------------------------------------- | ---------------------------------------------------------------------------------------------- |
| スケジュール付きパイプライン、バックフィル、SLA、多数のサービス横断依存 | Cloud Composer                         | [[Orchestration/Workflows\|Workflows]] | Workflowsは軽量なAPIチェイニング向けで、DAGスケジューリングやバックフィル運用のための設計ではない |
| シンプルなステップ調整（HTTP/GCP API呼び出し）、イベント駆動のつなぎ    | [[Orchestration/Workflows\|Workflows]] | Cloud Composer                         | Composerは運用負荷が重い。リトライ/分岐付きの少数API呼び出し程度では選ばない                    |

## セキュリティとアクセス制御
- 環境サービスアカウントには最小権限の [[Security/IAM|IAM]] ロールを付与する。
- 可能ならプライベートIP環境を優先する。
- 認証情報はすべて [[Security/Secret-Manager|Secret Manager]] に保存する（DAGコードや平文Airflow接続に置かない）。
- Composer web serverをプライベート化する場合は、パブリックIPではなくPSC（プライベートエンドポイント）を使う（公開は制約違反）。

## 運用と信頼性
- DAGは小さく保ち、パース時の重いimportを避ける（パースが遅いとスケジューラ全体が遅延する）。
- ワーカー枯渇を避けるため、妥当なconcurrency/parallelism制限を設定する。
- タスク時間、キュー時間、スケジューラのheartbeatを監視する。
- 一時障害には指数バックオフ付きリトライを使う。
- 本番ではSequential executorより、CeleryまたはKubernetes executorを優先する。

## よくある落とし穴
- Airflowワーカー内で重い計算を実行する — ワーカーはオーケストレーション専用。`DataflowCreateJobOperator`、`DataprocSubmitJobOperator`、`BigQueryInsertJobOperator` にオフロードする。
- パース時の重いPython importや動的DAG生成 — スケジューラは周期ごとに全DAGファイルを再パースする。遅いパースは環境全体を遅延させるため、モジュールレベルのコードを最小にする。
- リトライ対象タスクに冪等性がない — Airflowは自動で再実行する。重複を避けるため、書き込みは `WRITE_TRUNCATE`、`MERGE`、パーティション上書きを使う。
- `mode='poke'` のsensorを短い間隔で使う — 待機中ずっとワーカースロットを占有する。`mode='reschedule'` でチェック間にスロットを解放する。
- テストなしで `catchup=True` を有効化する — `start_date` 以降の未実行間隔をすべてリプレイし、下流を飽和させうる。`max_active_runs` で制限し、短い期間で先にテストする。
- 認証情報をDAGコードや平文Airflow接続に保存する — シークレットはmetadata DBとweb UIで見える。認証情報はすべて [[Security/Secret-Manager|Secret Manager]] に保存し、Secrets Backendで参照する。
- 本番でSequential executorを使う — 1タスクずつ実行され環境全体が直列化される。実運用ではCeleryまたはKubernetes executorを使う。

## 連携
- [[Storage/BigQuery|BigQuery]]: scheduled queries, dataset maintenance, and monitoring.
- [[Storage/Cloud-Storage|Cloud Storage]]: file arrival sensors and staging data.
- [[Processing/Dataflow|Dataflow]]: launch streaming or batch pipelines.
- [[Processing/Dataproc|Dataproc]]: submit Spark jobs for heavy processing.
- [[Ingestion/PubSub|Pub/Sub]]: event-driven triggers and notifications.
- [[Security/Secret-Manager|Secret Manager]]: store and retrieve connection secrets.

## クイックチェックリスト
- [[Storage/BigQuery|BigQuery]] と [[Storage/Cloud-Storage|Cloud Storage]] に揃えてリージョンを選ぶ。
- DAGの命名規約とオーナーを定義する。
- 接続とシークレットを [[Security/Secret-Manager|Secret Manager]] に設定する。
- タスク失敗とSLAミスのアラートを構成する。
- パイプラインごとにリトライとバックフィル戦略を文書化する。
- タスクの冪等性を強制する（パーティション上書き、MERGE、WRITE_TRUNCATE）。
