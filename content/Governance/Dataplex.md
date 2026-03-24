# Dataplex

Dataplex は、[[Cloud-Storage|Cloud-Storage]] と [[Storage/BigQuery|BigQuery]] をまたいでデータ管理を統合する、GCPのデータファブリックサービスである。データを移動せずに、データレイク/ウェアハウスに対して中央集約のガバナンス、メタデータ、データ品質を提供する。

## ユースケース
- 複数のデータレイク/ウェアハウスに一貫したガバナンスとメタデータを適用する。
- 発見（discovery）とコンプライアンスのために、データ資産をカタログ化/分類する。
- データ品質ルールと監視を標準化する。
- 明確なオーナーシップとアクセスパターンでデータドメインを整理する。

## メンタルモデル
- Dataplexはデータを保存せず、インプレースで管理する。
- データロケーションを登録し、その上にDataplexがガバナンスを重ねる。
- Dataplexはraw zoneのデータを自動発見/カタログ化し、curated zoneパイプライン向けにガバナンス/品質シグナルを付与できる。変換（transform）は [[Processing/Dataflow|Dataflow]] / [[Processing/Dataproc|Dataproc]] / [[Storage/BigQuery|BigQuery]] の責務。
- assetは特定パイプラインではなく、lakeとzoneに紐づく。
- Dataplexは [[Storage/BigQuery|BigQuery]] と [[Cloud-Storage|Cloud Storage]] を置き換えるのではなく補完する。

## コア概念
- Lake：データドメインの最上位コンテナ。
- Zone：lake内の論理グループ（raw/curated/sandbox）。
- Asset：zoneに登録されたデータリソース（[[Cloud-Storage|GCS]] バケット または [[Storage/BigQuery|BigQuery]] データセット）。
- Entity：カタログが認識するテーブル、またはファイルベースデータセット。
- Catalog：メタデータ層（スキーマ、タグ、分類）。

## ガバナンスとカタログ
- asset/entityの中央集約メタデータ。
- 分類（PII、機微度、オーナー）向けのタグテンプレート。
- アクセスポリシーとdiscovery制御。
- 利用可能な場合はリネージ統合（[[Data-Catalog|Data Catalog]] 経由）。
- Dataplexは [[Cloud-Storage|GCS]] と [[Storage/BigQuery|BigQuery]] 全体にガバナンスを適用する。一方、[[Data-Catalog|Data Catalog]] はdiscovery/メタデータ専用。

## リネージとDiscovery（主要機能）
- 検索可能なメタデータによる、プロジェクト/サービス横断の統合discovery。
- 対応範囲では自動リネージでエンドツーエンド可視化。
- 組み込みのデータ品質スキャンとプロファイリングによる、迅速なマネージド検証。

## データ品質
- ルール（NULLチェック、レンジ、スキーマ制約）を定義する。
- スキャンをスケジュールし、品質を時系列で追跡する。
- スコアカードでデータ健全性を可視化する。

## セキュリティとアクセス制御
- lake/zone/asset へのアクセスは [[Security/IAM|IAM]] ベース。
- 下位ストレージ（[[Cloud-Storage|GCS]]/[[Storage/BigQuery|BigQuery]]）のアクセス制御を継承する。
- 列レベルセキュリティには、[[Storage/BigQuery|BigQuery]] のpolicy tagsを使う。

## 運用と監視
- スキャン、ジョブ、データ品質結果を監視する。
- asset変更とスキーマドリフトを追跡する。
- 品質ルール失敗にアラートを使う。

## よくある落とし穴
- オーナー不在のままassetを登録する — 無主のassetはガバナンス不能になる。品質、アクセスレビュー、コンプライアンスの責任者が不在になるため、登録時にオーナーを割り当て、ポリシーとして強制する。
- DataplexがIAMと独立にアクセスを強制できると誤解する — Dataplexは既存IAMの上にガバナンスを重ねる。下位のGCSバケット/BigQueryデータセットへのアクセスは、依然としてIAMが直接制御し、Dataplexポリシーだけではread/writeを制限できない。
- zone設計を省き、rawとcuratedを混在させる — raw/curated分離がないと未検証データが下流を汚染する。zone構造があって初めて、品質ルールとリネージが意味を持つ。
- Dataplexを変換/処理ツールとして扱う — Dataplexはインプレースで発見/カタログ化/ガバナンスするが、transformは実行しない。処理は [[Processing/Dataflow|Dataflow]] / [[Processing/Dataproc|Dataproc]] / [[Storage/BigQuery|BigQuery]] の責務。
- 失敗アラートなしで品質ルールを定義する — スキャンは結果を記録するだけで、アラートがないと何も起きない。品質ルール失敗時に通知し、責任オーナーへ自動的に届くようにする。
- Dataplex と [[Governance/Data-Catalog|Data Catalog]] を混同する — Data Catalogは個別資産の発見/タグ付け/メタデータ検索。Dataplexはlakeレベルのガバナンス、zone管理、データ品質スキャン、ドメイン全体の統合ポリシーを提供する。

## 連携
- [[Cloud-Storage|Cloud-Storage]]: lake storage assets.
- [[Storage/BigQuery|BigQuery]]: warehouse datasets as assets.
- [[Processing/Dataflow|Dataflow]] / [[Processing/Dataproc|Dataproc]]: processing pipelines that write into Dataplex-managed zones.
- [[Security/IAM|IAM]]: access control foundation.

## BigLake 
BigLake は、[[Cloud-Storage|Cloud Storage]] と [[Storage/BigQuery|BigQuery]] のデータに対し、一貫したアクセス制御とガバナンスを提供する統合ストレージレイヤーである。データを移動せずに、BigQuery風の権限ときめ細かなアクセスをレイク/ウェアハウス横断で適用できる。

## クイックチェックリスト
- lakeとzones（raw/curated/sandbox）を定義する。
- assetを登録する（[[Cloud-Storage|GCS]] バケット、[[Storage/BigQuery|BigQuery]] データセット）。
- タグ/分類とオーナーシップメタデータを付与する。
- データ品質ルールと監視を設定する。
- 下位ストレージのアクセスと [[Security/IAM|IAM]] を整合させる。
