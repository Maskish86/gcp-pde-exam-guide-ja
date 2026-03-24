# Cloud Storage (GCS)

Google Cloud Storage は、GCPのオブジェクトストレージサービスである。分析パイプライン（[[Processing/Dataflow|Dataflow]]、[[Processing/Dataproc|Dataproc]]、[[Storage/BigQuery|BigQuery]]、Vertex AI）における標準のランディングゾーン/データレイク層。

## ユースケース
- [[Storage/BigQuery|BigQuery]] にロードする前のrawファイル（エクスポート、CDCダンプ、ログ）の着地点。
- バックフィル、監査、再処理のための不変（immutable）かつリプレイ可能なデータの保管。
- 標準形式（Parquet/Avro）でシステム間データを受け渡す。
- 下流ジョブ向けの大きな成果物（モデルファイル、埋め込み、特徴量エクスポート）の保管。

## メンタルモデル
- バケットはオブジェクトを保持する。「フォルダ」は実ディレクトリではなくプレフィックスである。
- インプレースのリネームはできない（move = copy + delete）。
- 強整合：read/list は直近の書き込みを即時に反映する（write-then-read パイプラインに安全）。
- レイテンシとエグレスコストを避けるため、ストレージとコンピュートは同一リージョンに揃える。

## コア概念
- **バケット**: 最上位コンテナ（グローバル一意の名前）。
- **オブジェクト**: バイト列 + メタデータ（名前、content type、カスタムフィールド）。
- **プレフィックス**: 疑似ディレクトリの慣習（例：`raw/dt=2026-02-05/...`）。
- **Generation**: オブジェクトの世代（バージョン）ID（安全な並行書き込みのための条件指定に使う）。

## ロケーション
| 種別 | 挙動 |
| --- | --- |
| Region | 単一リージョン — 同一リージョンのコンピュートと組み合わせると最小レイテンシ |
| Dual-region | 2リージョン — 高可用。重要データに推奨 |
| Multi-region | 広域 — 新規設計ではDual-regionに置き換わりつつある |

## ストレージクラス
ストレージクラスはアクセス速度ではなく、価格と取り出しコストに影響する。

| クラス    | アクセス頻度        | 補足                                       |
| -------- | ----------------- | ------------------------------------------- |
| Standard | 頻繁              | 取り出しコストなし                           |
| Nearline | 月次程度           | 取り出しコストあり（最小保管30日）            |
| Coldline | 四半期程度         | 取り出しコストが高い（最小保管90日）          |
| Archive  | 稀/コンプライアンス | 取り出しコスト最大（最小保管365日）           |

ライフサイクルルールで、経過日数に応じて自動でクラス移行や削除を行う。

## データレイクのレイアウトと形式
命名規約を、パーティションとオーナーシップの境界として扱う。

**ゾーン構成:**
- `landing/` — 不変のソース投下（write-once）
- `raw/` — ソース準拠、追記専用（append-only）
- `staging/` — 中間出力（短期保持）
- `curated/` — 分析可能なデータセット（analytics-ready）

**時間パーティション風のプレフィックス例:**
`raw/source=myapp/dt=2026-02-05/part-0000.parquet`

**形式の指針:**
- Parquet / ORC — カラムナ。分析と [[Storage/BigQuery|BigQuery]] ロードに最適
- Avro — 行指向。スキーマ進化に強く、ストリーミングに向く
- JSON / CSV — 簡単だがサイズが大きく遅い。必要な場合のみ

## データレイク・アーキテクチャ

```mermaid
flowchart TB

subgraph SRC["ソース"]
    APP[アプリケーション]
    CDC["運用DB<br/>(変更データキャプチャ(CDC)・抽出)"]
    EXT[外部 / オンプレミス]
end

subgraph GCS["GCS - データレイクゾーン"]
    LAND["landing/<br/>(書き込み1回)"]
    RAW["raw/<br/>(追記専用)"]
    STG["staging/<br/>(短期保持)"]
    CUR["curated/<br/>(分析向け)"]
    LAND ~~~ RAW ~~~ STG ~~~ CUR
end

PS[Pub/Sub]
ORC["Composer"]

subgraph PROC["処理"]
    DF[Dataflow]
    DPC[Dataproc]
end

BQ[BigQuery]

%% Sources -> landing
APP -->|ファイル投下| LAND
CDC -->|抽出| LAND
EXT -->|一括転送| LAND

%% Event-driven ingestion
LAND -.->|オブジェクト確定| PS
PS -.->|トリガー| DF
DF -->|検証 · タグ付け| RAW

%% Orchestrated batch: raw -> staging
ORC -.->|スケジュール| DF
ORC -.->|スケジュール| DPC
RAW -->|ソース| DF
RAW -->|Sparkソース| DPC
DF -->|出力| STG
DPC -->|出力| STG

%% Curated -> analytics
STG -->|昇格(本番反映)| CUR
CUR -->|ロードジョブ| BQ
CUR -.->|外部テーブル| BQ
```

## セキュリティとアクセス制御

**バケットのデフォルト設定:**
- **Uniform Bucket-Level Access (UBLA)** を有効化 — オブジェクトACLを無効化し、IAMのみに統一する。
- **Public Access Prevention** を有効化 — 誤って公開してしまう事故を防ぐ。
- 機微度（raw vs curated vs temp）でバケットを分離し、IAM付与を単純に保つ。

**よく使うIAMロール:**

| ロール                         | 付与される権限                                                                  |
| ----------------------------- | ----------------------------------------------------------------------------- |
| `roles/storage.objectViewer`  | オブジェクトの読み取りと一覧（書き込み不可）                                   |
| `roles/storage.objectCreator` | オブジェクトの書き込みのみ（読み取り/一覧不可）                               |
| `roles/storage.objectAdmin`   | オブジェクトの読み書き/削除（バケット設定/IAMは不可）                          |
| `roles/storage.admin`         | バケットとオブジェクトのフル管理（IAM、ライフサイクル、保持）— 付与は最小限にする |

**広いIAM付与なしで共有する:**
- Signed URLs — 特定オブジェクトへの期限付きアクセス。

**暗号化:**
- 既定：Google管理の暗号化（自動）。
- CMEK：鍵管理、ローテーション、監査要件のために [[Cloud-KMS|Cloud KMS]] を使う。
  - バケットに設定したCMEKは **新規書き込みのみ** に適用される。既存オブジェクトは明示的に再書き込みが必要。
  - 鍵が侵害された場合：新しい鍵を作成 → バケット既定鍵として設定 → オブジェクトをコピー/再書き込み（通常は新しいバケットへ）。

**プライバシーとコンプライアンス:**
- 分析用バケットへコピーする前に、[[Security/DLP|DLP]] で機微フィールドを検出/マスクする。
- 制限付きrawバケットは、匿名化（de-identified）データセットと分離する。

**保持と復旧:**
- オブジェクトのバージョニング：上書き/削除時に古い世代を保持する（復旧に有用だが、放置するとコスト増）。
- 保持ポリシー/ホールド：最小保持期間を強制する（コンプライアンスに有用。削除をブロックする）。

## パイプライン信頼性パターン
- **冪等な書き込み**：決定的な出力名を使う、または一時プレフィックスへ書いてからアトミックにpromoteする。
- **並行性の安全**：preconditions（generation/metagenerationチェック）で、並行出力の上書きを防ぐ。
- **ファイルサイズ**：極小ファイルの爆発や、単一の巨大ファイルを避ける。分析ワークロードでは1ファイル数百MBを目安にする。

## ライフサイクルルール
- 条件：`age`、`matchesStorageClass`、`isLive`（バージョニング有効バケット）。
- **非対応**：コンテンツベース、拡張子、オブジェクトサイズでのフィルタ。
- よくあるパターン：`staging/` をN日後に削除、`raw/` を90日後にColdlineへ移行。
- バージョニングコストに注意：非current世代は、対象にするライフサイクルルールがないと静かに蓄積する。

## データの入出力（移動）

| ツール                              | 最適                                                                                                             |
| ---------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| `gcloud storage`                   | 既定CLI。今後はこちらが推奨                                                                                       |
| `gsutil`                           | レガシーCLI。非常に大きいファイルが少数の初期移行で有用（`-m` 並列、レジューム可能アップロード）                  |
| **Storage Transfer Service**       | マネージドでスケジュール可能な転送（on-prem → GCS、クロスクラウド、GCS → GCS）                                    |
| **Transfer Appliance**             | WANがボトルネックになるマルチPB移行向けのオフライン移行                                                            |
| **BigQuery Data Transfer Service** | 低運用負荷で、GCSからBigQueryへマネージドのバッチロード                                                            |

**Storage Transfer Service の注意点:**
- 分離環境または継続的なon-prem取り込みでは、プライベート接続（VPN / Interconnect + Private Google Access）と **STS agent** を使う。
- 定期的なPOSIX/NFS同期では、STS agentが増分コピー（変更ファイルのみ）に対応し、リトライ、チェックポイント、整合性チェックを備える。
- PB規模で時間制約が厳しい転送では、**Cloud Interconnect**（専用10/100Gbps）とSTS（並列ストリーム、リトライ、チェックサム検証、スケジューリング）を組み合わせる。

## 連携
- [[Storage/BigQuery|BigQuery]]：ロードジョブ（バッチ取り込み）、外部テーブル（その場クエリ。探索に有用だがネイティブより遅い）。
- [[Processing/Dataflow|Dataflow]] / [[Processing/Dataproc|Dataproc]]：主要なソース/シンク。リプレイ/バックフィルでは命名規約が効く。
- [[Ingestion/PubSub|Pub/Sub]] 通知：オブジェクトfinalizeでイベントを発行し、イベント駆動の取り込みを起動する。
- [[Processing/Data-Fusion|Data Fusion]]：バッチのファイル取り込み/ETLのドラッグ&ドロップUI。リアルタイム異常検知には不向き（[[Ingestion/PubSub|Pub/Sub]] + [[Processing/Dataflow|Dataflow]] のストリーミングを使う）。

## 運用とコスト制御
- ライフサイクルルールが主要なコストレバー（クラス移行 + temp/staging削除）。
- Cloud Audit Logs：コンプライアンスのために、ポリシー/アクセス変更を記録する。
- Cloud Monitoring / Logging：バケット増加、エラー率、アクセスパターンを追跡する。

## クイックチェックリスト
- UBLAを有効化し、Public Access Preventionを強制する。
- 最小権限IAM（グループ/サービスアカウント、機微度でバケット分離）。
- 命名規約とゾーン構成を文書化する（時間パーティションのプレフィックスを含む）。
- ライフサイクルルールを設定する（temp削除、クラス移行、バージョニング世代の期限）。
- バージョニング/保持の方針を意図的に決め、コストを監視する。
