# Cloud Logging

Cloud Logging は、GCPサービスとアプリケーションのログデータを収集・保存・分析する。検索可能なログ、ログベースメトリクス、長期保持や分析のためのsinkへのルーティングを提供する。

## ユースケース
- 失敗したパイプラインやジョブエラーをデバッグする。
- タスクログからデータ鮮度問題を追跡する。
- アラート用にログベースメトリクスを作る。
- コンプライアンスのため、ログをストレージ/分析システムへルーティングする。

## メンタルモデル
- ログはLog Explorerへ取り込まれ、プロジェクト単位で保存される。
- ログは保持/分析のためにsink（BigQuery/Cloud Storage/Pub/Sub）へルーティングできる。
- ログベースメトリクスは、ログパターンを時系列へ変換し、アラートに使える。
- severityとlabelsでノイジーなストリームをフィルタできる。

## コア概念
- Log entry：timestamp/severity/payloadを持つ単一イベント。
- Log bucket：プロジェクト/組織内のログ保存先。
- Log sink：他の宛先へのエクスポート経路。
- Log-based metric：ログフィルタから生成されるメトリクス。

## よくあるパターン
- DataflowジョブログをBigQueryへエクスポートして分析する。
- 重要DAGのエラー件数にアラートを作る。
- パイプラインステップと行数に構造化ログを使う。
- aggregated sinksでプロジェクト横断のログを集約する。

## 連携
- [[Cloud-Monitoring|Cloud-Monitoring]]: alert on log-based metrics.
- [[Processing/Dataflow|Dataflow]]: job and worker logs.
- [[Processing/Dataproc|Dataproc]]: cluster and job logs.
- [[Storage/BigQuery|BigQuery]]: log export destination.
- [[Cloud-Storage|Cloud Storage]]: log archive destination.
- [[Ingestion/PubSub|Pub/Sub]]: stream logs to downstream consumers.

## セキュリティとアクセス制御
- ログアクセスには最小権限の [[Security/IAM|IAM]] ロールを使う。
- PII/シークレットを含む機微ログへのアクセスを制限する。
- 機微度ごとに保持先を分けるため、log sinksを使う。

## 運用と信頼性
- パイプラインステップのログ形式を標準化する。
- 監査とポストモーテムに十分な期間ログを保持する。
- labelsで環境とパイプラインバージョンを追跡する。

## よくある落とし穴
- マスキングせずに機微データをログ出力する。
- 明示的メトリクスを出さず、ログだけでメトリクス代用する。
- sinkやフィルタなしにログ量が増え、コストが想定外になる。

## クイックチェックリスト
- 環境とコンプライアンス要件でログ保持を定義する。
- 長期保存/分析のためのsinkを作成する。
- 重要エラーパターンのログベースメトリクスを追加する。
- オンコール向けにログ検索フィルタを文書化する。
- 四半期ごとにログ量とコストをレビューする。
