# AlloyDB

AlloyDB は、PostgreSQL互換で高性能に最適化された、GCPのマネージド データベースである。標準的なPostgreSQLより高いスループットと低レイテンシを必要とする運用ワークロードを対象にしつつ、PostgreSQLのツール/ドライバ互換を維持する。

## ユースケース
- 高い読み取り/書き込みスループットが必要なOLTPシステム。
- 再プラットフォームなしで **より高いスループット** または **HTAP**（混在: OLTP + analytics）が必要なら、**Cloud SQL for PostgreSQL より AlloyDB** を選ぶ。
- 運用データに対して分析クエリが頻繁に走るハイブリッドワークロード。
- マネージドでスケールしつつPostgreSQL互換を求めるアプリケーション。
- 分析基盤へのCDCソース（[[Ingestion/Datastream|Datastream]] 経由）。

## メンタルモデル
- 計算とストレージを分離した、PostgreSQL互換エンジン。
- **HTAP = OLTP + analytics を1つのシステムで**（row engine + columnar accelerator）: ETL/二重ストアの複雑性を減らす。
- Primary + read pool により、読み取りと分析クエリをスケールする。
- ストレージは分散かつマネージドで、コンピュートは独立にスケールできる。
- 多くのPostgreSQL拡張とドライバに互換。

## コア概念
- Cluster: primary インスタンス +（任意で）read pool。
- Primary: 書き込みとトランザクションを処理する。
- Read pool: 読み取り専用ノードでクエリをスケールする。
- Backups: ポイントインタイムリカバリ（PITR）付きの自動バックアップ。

## 性能とスケーリング
- read poolノード追加で読み取りをスケールする。
- 書き込みが重いワークロードはコンピュートを垂直スケールする。
- ホットデータをメモリに載せ、キャッシュヒット率を監視する。

## 可用性と信頼性
- 自動フェイルオーバーを伴う高可用性。
- リージョンデプロイ。必要に応じてレプリカで耐障害性を高める。
- 障害復旧にはバックアップとPITRを使う。

## セキュリティとアクセス制御
- リソースアクセス/管理にはIAMを使う。
- 可能なら内部アクセスにprivate IPを使う。
- 暗号化制御のため、[[Cloud-KMS|Cloud KMS]] によるCMEKをサポートする。

## よくある落とし穴
- ウェアハウスのように扱う — 依然としてOLTPデータベースである。
- 書き込みが重いのにprimaryのコンピュートを過小サイジングする。
- 分析/読み取りをread poolへ分離しない。

## 連携
- [[Ingestion/Datastream|Datastream]]: AlloyDBから分析基盤へのCDC。
- [[Processing/Dataflow|Dataflow]]: AlloyDBから [[Storage/BigQuery|BigQuery]] へのETL。
- [[Security/IAM|IAM]] と [[Cloud-KMS|Cloud KMS]]: アクセス/暗号化制御。

## クイックチェックリスト
- primaryのサイズとread pool数を決める。
- private IP（またはセキュアな接続方式）を選ぶ。
- バックアップを構成し、PITRを復元テストする。
- 要件があればIAMとCMEKを設定する。
- 分析レプリケーションが必要ならCDCを計画する。
