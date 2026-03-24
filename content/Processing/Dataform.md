# Dataform

Dataform は、[[Storage/BigQuery|BigQuery]] 向けのデータ変換フレームワークである。SQLベースのパイプライン定義、依存関係管理、バージョン管理下での増分ビルド実行を可能にする。

## ユースケース
- raw/staging テーブルからキュレート済みデータセットを構築する。
- 依存関係グラフでSQL変換を管理する。
- 反復実行可能な増分モデルを実装する。
- ガバナンスされた運用で再利用可能なデータモデルを共有する。

## メンタルモデル
- Dataform はSQLをコンパイルし、操作のDAGとして扱う。
- run はBigQuery上のSQL実行のオーケストレーションであり、別コンピュートではない。
- 増分テーブルは、正しく設計すればフルリビルドを避けられる。
- 環境（dev/prod）はデータセットと権限を分離すべき。

## コア概念
- Project：モデルと設定の集合。
- Model：SQL変換（table/view/incremental）。
- Assertions：データ品質チェック（一意性、NULL、制約）。
- Dependencies：モデル間の明示的な参照。
- Runs：BigQuery上でコンパイル済みグラフを実行すること。

## よくあるパターン
- 明確な命名による Raw -> staging -> curated のレイヤリング。
- `uniqueKey` と `updatePartitionFilter` を使った増分モデル。
- 重要な業務ルールと鮮度に対するAssertions。
- pull request とコードレビューによるバージョン管理。

## 一意性とNULLのAssertions
- データ品質チェックには、組み込みAssertions（`uniqueKey`、`nonNull`、またはカスタム）を使う。
- Assertions はDataformパイプラインの一部として実行され、違反があるとrunを失敗させる。
- 外部DQツールや手動UDFチェックと比べて、最も効率的でネイティブなアプローチである。

## 連携
- [[Storage/BigQuery|BigQuery]]: 実行エンジン兼ストレージ。
- [[../Security/IAM|IAM]]: サービスアカウント/ユーザーの権限。
- [[Cloud-Composer|Cloud-Composer]] または [[../Orchestration/Workflows|Workflows]]: runのスケジュール/トリガー。

## セキュリティとアクセス制御
- Dataformのサービスアカウントには最小権限の [[../Security/IAM|IAM]] を使う。
- 誤書き込みを防ぐため、dev/prodのデータセットを分離する。
- シークレットの埋め込みを避け、可能ならマネージド接続を使う。

## 運用と信頼性
- 大規模テーブルは増分ビルドでコストを制御する。
- 失敗を監視し、スケジュール実行にアラートを設定する。
- 保守性向上のため、SQLスタイルを一貫させる。

## よくある落とし穴
- 安定した一意キーなしで増分モデルを使う。
- 1つのデータセットに複数環境を混在させる。
- 失敗するまでAssertionsを無視する。

## クイックチェックリスト
- データセットの命名規約（raw/staging/curated）を定義する。
- 環境と権限を設定する。
- 重要テーブルにAssertionsを追加する。
- 必要に応じて増分ロジックを実装する。
- runのスケジュールとオーナーを文書化する。
