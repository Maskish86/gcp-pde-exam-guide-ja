# Dataprep

Cloud Dataprep（Trifacta提供）は、[[Cloud-Storage|Cloud Storage]] または [[Storage/BigQuery|BigQuery]] のデータを対象に、プロファイリング、クリーニング、変換を行うマネージドなビジュアル データ準備ツールである。実行は内部的に [[Processing/Dataflow|Dataflow]] ジョブを生成して行い、Dataprep専用の実行エンジンは存在しない。

## メンタルモデル
- Dataprep は **Dataflow上のUI** — パイプライン作成を、ビジュアルなrecipeエディタに抽象化する。
- エンジニアではなく、**アナリスト/データスチュワード** 向け。
- 実行は **常にバッチ** — GCSデータはDataflow、BigQueryデータはBigQueryで実行する。
- 本番パイプラインにおけるDataflowやData Fusionの代替ではない。

## 試験ドメイン
- データ処理システムの設計
- データ処理システムの構築と運用化
- ソリューション品質の確保

## コア概念

| 概念        | 説明                                                    |
| ----------- | ------------------------------------------------------- |
| Flow        | Dataprepプロジェクト（datasets/recipes/outputs を接続） |
| Recipe      | クリーニング/変換ステップの順序付きシーケンス           |
| Transformer | recipeを作るためのビジュアルエディタ                    |
| Dataset     | GCSまたはBigQueryのソースデータ                         |
| Job         | flowの実行（Dataflowジョブとして動く）                  |

## 制約
- レイテンシ: 対話的/アドホック準備向け。低レイテンシのストリーミングではない。
- スケール: 中程度（実行は [[Processing/Dataflow|Dataflow]] または [[Storage/BigQuery|BigQuery]] にオフロード可能）。
- 一貫性: バッチで決定的な変換。
- コスト: 下位の実行エンジン課金が中心。SQLだけの些細な変換には不向き。
- 運用負荷: 低（マネージドなビジュアルツール）。
- バッチ vs ストリーミング: **バッチのみ**

| サービス           | 適する条件                                                   | 失敗する条件                                                   |
| ------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **Dataprep**       | ビジュアル クリーニング、アナリスト主導の準備、最小限のコーディング | ストリーミングパイプライン、複雑な多段エンジニアリングワークフロー |
| **Dataflow**       | カスタム変換での大規模バッチ/ストリーミング                  | GUIが必須、素早いワンオフ クリーニング                         |
| **Data Fusion**    | コネクタが豊富でガバナンスされた再利用可能ETL                | アドホック準備、小規模ジョブにはオーバーヘッドが大きい        |
| **BigQuery SQL**   | BigQueryデータに対するSQLのみの変換                           | ファイル中心の準備、非SQLクリーニングが必要                   |

## 正しい選択
- 制約が「ファイル/BigQueryテーブルに対する **低いエンジニアリング負荷 + ビジュアル準備**」で、バッチ実行が許容されるなら **Dataprep**。
- カスタムロジックやストリーミングが必須なら **Dataflow**。
- 本番ETLでコネクタの幅とガバナンスが必要なら **Data Fusion**。
- BigQueryでSQLだけの変換なら **BigQuery SQL**。

## 他を選ばない理由
- **Dataflow:** アナリスト主導の準備には過剰で、開発/運用負荷が高い。
- **Data Fusion:** 単純/ワンオフのクリーニングには運用/コストが重い。
- **BigQuery SQL:** 非SQLの準備やファイル中心ワークフローに弱い。

## よくある試験の罠
- 「アナリストによるビジュアルなセルフサービス クリーニング」→ Dataflow/Data Fusionではなく **Dataprep**。
- 「Dataprepの実行エンジン」は存在しない → **Dataflowジョブを生成して実行する**。コスト = Dataflowワーカー時間 + Dataprepサービス料金。
- **ストリーミング** にDataprepを選ぶ → **バッチのみ**。リアルタイムはDataflowまたはPub/Sub。
- **複雑で多段な本番ETL** にDataprepを選ぶ → **Data Fusion**（コネクタ + ガバナンス）または **Dataflow**（カスタム変換）。
- ソースが **GCSまたはBigQuery以外** なのにDataprepを選ぶ → DataprepはGCS/BigQueryしか読めない。他ソースはDataflowまたはData Fusion。
- 「SQLだけのBigQuery変換でコスト最小」→ Dataprepではなく **BigQuery SQL**。Dataprepは純SQLでも不要なDataflowワーカーのオーバーヘッドを足しやすい。
