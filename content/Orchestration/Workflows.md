# Workflows

Workflows は、APIとクラウドサービスを調整するためのGCPサーバレスオーケストレーションサービスである。サーバ管理なしに、YAML/JSONで定義したステップをリトライ、条件分岐、並列分岐つきで実行する。

## ユースケース
- マネージドサービスをまたぐ軽量パイプラインをオーケストレーションする。
- 取り込み、検証、通知のためにAPIをつなぐ。
- Pub/Sub やHTTPでトリガーされるイベント駆動フローを実行する。
- シンプルなスクリプトを、監査可能なマネージドワークフローに置き換える。

## メンタルモデル
- 各ステップはAPI呼び出し、ロジック実行、待機を行い、状態はWorkflowsが管理する。
- Workflowsはオーケストレーションであり、重い計算処理ではない。
- リトライとエラーハンドリングは、ステップ単位またはワークフロー単位で定義する。
- 長時間/重い処理は [[Processing/Dataflow|Dataflow]] などへオフロードする。

## コア概念
- Workflow：定義とデプロイの単位。
- Step：単一のアクション（call、assign、switch、try/catch）。
- Execution：ワークフローの1回の実行。
- Connector：GCP API向けの組み込み連携。

## よくあるパターン
- Cloud Storageを呼ぶ → 検証 → BigQueryロードを起動する。
- API呼び出しを並列ファンアウトし、結果を集約する。
- 一時障害は指数バックオフ付きでリトライする。
- 共通ロジックはsubworkflowsに切り出す。

## 複雑ロジックのオフロード（Cloud Run）
- Workflowsの標準ライブラリを超える複雑な業務ロジックは、Cloud Run（またはCloud Functions）で実装する。
- 小さなAPIペイロードが適する。Workflowsがオーケストレーションし、Cloud Runが計算する。
- 速い起動と自動スケールのフルマネージド実行環境により、解をシンプルに保てる。
- ステートレスなHTTPコンピュートにはCloud Runを選ぶ。クラスタレベルの制御やカスタムランタイムが必要な場合のみGKEを優先する。

## 連携
- [[Cloud-Storage|Cloud Storage]]: ファイル可用性のトリガー/検証。
- [[Storage/BigQuery|BigQuery]]: ジョブの開始と結果確認。
- [[Processing/Dataflow|Dataflow]]: バッチ/ストリーミングジョブの起動。
- [[Ingestion/PubSub|Pub/Sub]]: イベントトリガーと通知。
- [[Secret-Manager|Secret Manager]]: APIキーとトークンの保管。

## 試験の判断：Workflows vs Composer vs Cloud Run
| 要件シグナル | 選ぶ | 却下 | 却下理由（ディストラクタ排除） |
|---|---|---|---|
| 軽量なAPIオーケストレーション（リトライ/分岐/並列呼び出し） | Workflows | Cloud Composer | Composerは重い。DAGスケジューリング/バックフィルが不要なら選ばない。 |
| スケジューリング + バックフィル + ジョブ間の多数依存 | Cloud Composer | Workflows | Workflowsは「つなぎ」に強いが、Airflow風のスケジューラ/バックフィル運用ツールではない。 |
| HTTPの背後にあるカスタム計算/業務ロジック | Cloud Run | Workflows | Workflowsは汎用コンピュート実行環境ではない。重いロジックではなく、調整に使う。 |

## セキュリティとアクセス制御
- ワークフローのサービスアカウントには最小権限の [[Security/IAM|IAM]] ロールを付与する。
- ワークフロー定義にシークレットを埋め込まず、[[Secret-Manager|Secret Manager]] を使う。
- 内部サービス呼び出しではプライベート接続を優先する。

## 運用と信頼性
- 構造化ログを使い、correlation IDを含める。
- 実行（execution）を監視し、失敗率にアラートを設定する。
- 保守性のため、ワークフローを小さく分割して合成可能に保つ。

## よくある落とし穴
- 専用サービスにオフロードせず、ワークフロー内で重い計算を行う。
- 冪等性を無視し、リトライで重複アクションが発生する。
- 設定を使わずにエンドポイントをハードコードする。
- `kubectl scale` をオートスケールと誤解する（HPAまたはCloud Run autoscaleを使う）。

## クイックチェックリスト
- 入出力を明確に定義する。
- 外部呼び出しにリトライとタイムアウト処理を追加する。
- シークレットを [[Secret-Manager|Secret Manager]] に保存する。
- 重要ステップとエラーにログを追加する。
- [[Cloud-Monitoring|Cloud Monitoring]] と [[Cloud-Logging|Cloud Logging]] で監視する。
