# Data Catalog

Data Catalog は、GCPのマネージド メタデータ/ディスカバリサービスである。データを移動せずに、BigQueryやCloud Storageなどのサービス横断でデータ資産を見つけ、記述し、タグ付けし、ガバナンスするのに役立つ。

## ユースケース
- データセット/テーブル/ファイルの検索可能な棚卸しを作る。
- ビジネス/技術メタデータ（オーナー、機微度、定義）を付与する。
- タグとpolicy tagsでガバナンスを標準化する。
- アナリストや下流チームのdiscoverabilityを高める。

## メンタルモデル
- Data Catalogが保存するのはメタデータのみで、データ本体は保存しない。
- entryが資産を表し、tagが意味とガバナンス文脈を付与する。
- policy tags がBigQueryの列レベルアクセス制御を駆動する。
- [[Governance/Dataplex|Dataplex]] のようなツールを置き換えるのではなく補完する。
- [[Storage/BigQuery|BigQuery]]、[[Cloud-Storage|Cloud Storage]]、[[OperationalDBs/Bigtable|Bigtable]]、[[Ingestion/PubSub|Pub/Sub]] から **メタデータを自動抽出** する（カスタムスクリプト不要）。
- Compute Engine上のPostgreSQLはマネージドメタデータソースではないため自動スキャンできない。コネクタを使う。

## コア概念
- Entry：データ資産（テーブル、データセット、ファイルセット、またはカスタムentry）。
- Tag template：メタデータフィールド（owner、PII、SLAなど）のスキーマ。
- Tag：entryに付与されるメタデータ実体。
- Policy tag：BigQueryの列レベルセキュリティのタクソノミー。

## よくあるパターン
- データセットにownerとSLAをタグ付けし、説明責任を明確化する。
- 機微データ制御のために、列をpolicy tagsで分類する。
- ビジネスタグで資産をドメイン/プロダクトに紐づける。
- 対応サービスの自動取り込みでdiscoveryを最新に保つ。
- Cloud Storage移行では、[[Security/DLP|DLP]] でPIIを検出し、Data Catalogのタグで機微オブジェクトをラベル付けする。
- PII分析結果を構造化タグメタデータ（例：`contains_pii: true`、`pii_type: [name, dob]`）として保存し、後でクエリ/取得できるようにする。これは「検出/匿名化/暗号化」ではなくメタデータ保存の問題。

## 連携
- [[Storage/BigQuery|BigQuery]]: tables, datasets, policy tags.
- [[Cloud-Storage|Cloud Storage]]: file set metadata for lakes.
- [[Governance/Dataplex|Dataplex]]: catalog and governance at scale.
- [[Security/IAM|IAM]]: governs who can view and edit metadata.

## セキュリティとアクセス制御
- カタログアクセスには最小権限の [[Security/IAM|IAM]] を使う。
- タグの可視性を制御し、機微メタデータの露出を抑える。
- BigQueryの列レベルセキュリティはpolicy tagsで強制する。
- 列セキュリティは、policy tag access controlが有効で、かつユーザーが `roles/datacatalog.categoryFineGrainedReader` を持たない場合にのみ効く。
- 非公開タグテンプレートの閲覧には **tagTemplateViewer** が必要。テンプレートが **public** なら冗長。

## 運用と信頼性
- タグテンプレートをドメイン横断で一貫させる。
- 混乱を避けるため、古いタグはレビューして整理する。
- 重要資産のオーナーと定義を文書化する。

## よくある落とし穴
- タグを空のままにする/そもそも付与しない — タグなし資産はガバナンス/検索から見えない。空タグはノイズ。あとからではなく、導入時に共通テンプレートを強制する。
- チーム間でタグスキーマが不一致 — チームごとのテンプレート乱立で検索が分断され、ドメイン横断ガバナンスができない。オンボーディング前に正準テンプレートを合意する。
- カタログ可視性＝データアクセスと誤解する — Data Catalogはメタデータの閲覧/編集を制御するだけで、データ本体はBigQuery/GCS等のIAMで別途制御される。
- policy tags の `categoryFineGrainedReader` 制限を忘れる — policy tagsは、このroleが付与されていない場合にのみ列アクセスをブロックする。広範（大グループ/`allUsers` 等）に付与すると列制御が黙って無効化される。
- 非マネージドソースの自動スキャンを期待する — 自動抽出はBigQuery/GCS/Bigtable/Pub/Subが対象。カスタム/on-premソース（Compute Engine上PostgreSQL等）はコネクタか手動entryが必要。
- DLPスキャン結果をカタログタグへ書き戻さない — DLPはPII検出してもData Catalogを自動更新しない。発見結果を構造化タグ（`contains_pii: true`, `pii_type`）として明示的に書き込み、発見性と監査性を担保する。
- Data Catalog と [[Governance/Dataplex|Dataplex]] をレイク規模ガバナンスで混同する — Data Catalogはdiscovery/タグ付け。Dataplexはlake管理、データ品質ルール、zone横断の統合ガバナンスを提供する。

## クイックチェックリスト
- オーナー/機微度/SLAのタグテンプレートを定義する。
- 機微列をpolicy tagsで分類する。
- 主要データセットとレイクロケーションを登録する。
- メタデータオーナーとレビュー頻度を決める。
- チームがどう発見し、どうアクセス申請するかを文書化する。
