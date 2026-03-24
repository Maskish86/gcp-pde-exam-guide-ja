# IAM

Identity and Access Management（IAM）は、どのGCPリソースに対して誰が何をできるかを制御する。データプラットフォームにおける最小権限アクセスの中核。

## ユースケース
- データセット、バケット、パイプラインへのアクセスを付与する。
- 管理業務とデータアクセスを分離する。
- ジョブ/ワークフローのサービスアカウント権限を制御する。

## メンタルモデル
- ポリシーが、リソース上でprincipalとroleを紐づける。
- roleは権限（permissions）の集合。
- 既定は拒否。必要なものだけを付与する。

## コア概念
- Principal：ユーザー、グループ、サービスアカウント、またはドメイン。
- Role：事前定義またはカスタムの権限セット。
- Policy：リソース上の、principalとroleのbinding。
- リソース階層：org -> folder -> project -> resource。

## サービスアカウント
- 人ではなく、ワークロード/パイプラインに使う。
- 可能なら長期キーを避け、鍵ローテーションを行う。
- ワークロードごとに最小限のroleを付与する。

**認証情報の優先順位（上ほど推奨）:**
1. **Attached service account**（VM/Cloud Run/Dataflowジョブ）— キーファイルを作らない。認証情報は短命で自動ローテーションされ、メタデータサーバから取得される。基本的にこれが正解。
2. **Workload Identity Federation** — GCP外のワークロード向け。外部IDトークンを短命のGCP認証情報へ交換する。
3. **[[Secret-Manager\|Secret Manager]]** — キーファイルが本当に不可避な場合のみ。ハードコードよりは良いが、手動ローテーションが必要な長期認証情報である。

> サービスアカウントキーをSecret Managerに保存するのは、ハードコードより改善だが、通常の正解は「そもそもキーファイルを作らず、リソースにサービスアカウントを直接アタッチする」ことである。

## セキュリティとガバナンス
- 最小権限とリソース単位のbindingを使う。
- 管理roleとデータアクセスroleを分離する。
- 監査ログでアクセスとポリシー変更を監視する。
- VPC Service Controlsでサービスペリメータを作り、プロジェクト間のデータアクセスを遮断する。
- Cloud Logging：`roles/logging.viewer` は非公開のData Accessログを読めない。監査の可視性には `roles/logging.privateLogViewer` を使う。

## 設計のコツ
- 人間にはグループベースアクセスを優先する。
- 時間/リソース制約には条件付きIAMを使う。
- 本番でOwner/Editorのような過剰に広いroleを避ける。

## 連携
- [[Cloud-SQL|Cloud SQL]]：Auth proxy/connectors はセキュア接続のために `roles/cloudsql.client` を使う。
- [[Cloud-Storage|Cloud Storage]] / [[Storage/BigQuery|BigQuery]]：データセット/バケットのアクセス制御。
- [[Processing/Dataflow|Dataflow]] / [[Processing/Dataproc|Dataproc]]：ジョブ実行用サービスアカウント。

## よくある落とし穴
- サービスアカウントではなくユーザーに広いroleを付与する。
- 一時的なアクセスを剥がし忘れる。
- IAM制御なしにネットワーク許可リストだけに依存する。
- IAM条件だけでプロジェクト間のデータ持ち出し（exfiltration）が防げると誤解する。

## クイックチェックリスト
- リソース境界を定義する。
- 最小権限のroleを選ぶ。
- 個人ではなくグループ/サービスアカウントをbindingする。
- 監査ログ監視を有効化する。
