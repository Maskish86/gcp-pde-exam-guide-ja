# Secret Manager

Secret Manager は、APIキー、パスワード、証明書などの機微な設定を保存・管理する。バージョン管理されたシークレット、IAMベースのアクセス制御、監査ログ、必要に応じた [[Cloud-KMS|Cloud KMS]] 経由のCMEKを提供する。

## ユースケース
- パイプライン用のDB認証情報、APIトークン、サービスキーを保存する。
- ワークフロー（[[Processing/Dataflow|Dataflow]]、[[Processing/Dataproc|Dataproc]]、Cloud Run、Functions）のシークレットを一元管理する。
- バージョン管理された値で、コード変更なしにシークレットをローテーションする。
- 環境変数/ファイル/CIログへのシークレット拡散を減らす。

## メンタルモデル
- secret は名前付きコンテナで、値はバージョンとして管理される。
- アプリは特定バージョン（または "latest"）を参照する。
- アクセスはsecret単位でIAMにより制御される。
- レプリケーションが、シークレットデータの保存場所を制御する。

## コア概念
- Secret：リソース名とメタデータ。
- Secret version：不変のペイロード（enable/disable/destroy可能）。
- Replication：自動（マルチリージョン）またはユーザー管理（リージョン指定）。
- Labels：オーナー、コスト、ライフサイクル管理のためのメタデータ。

## アクセス制御とIAM
Common roles:
- `roles/secretmanager.secretAccessor` to read secret versions.
- `roles/secretmanager.admin` to manage secrets (use sparingly).

Design tips:
- Grant access to service accounts, not users.
- Separate readers from admins.
- Use least privilege by secret and environment (dev/prod).

## ローテーションとライフサイクル
- ローテーションは新しいversionを作成し、ロールバック用に旧versionを短期間だけ残す。
- 事故停止を避けるため、destroy前に旧versionをdisableする。
- スケジューラやCIでローテーションを自動化する。

## 暗号化とCMEK
- シークレットは既定で保存時暗号化される。
- 鍵の制御と監査性が必要なら、[[Cloud-KMS|Cloud KMS]] 経由でCMEKを使える。
- KMS鍵のリージョンは、secretのレプリケーション設定と揃える。

## 利用パターン
- 起動時にシークレットを読み、メモリにキャッシュする。
- シークレット値をログ出力したり、コマンドライン引数で渡したりしない。
- 変更がセンシティブならバージョン固定、オートローテーションなら "latest" を使う。

## 連携
- Cloud Run / Functions / GKE：実行時にシークレットを注入する。
- [[Processing/Dataflow|Dataflow]] / [[Processing/Dataproc|Dataproc]]：コネクタ/外部システム用にシークレットを参照する。
- [[Security/IAM|IAM]]：アクセスと職務分離を統制する。

## よくある落とし穴
- Secret Manager をバルクデータ暗号化に使う（代わりに [[Cloud-KMS|Cloud KMS]] を使う）。
- 旧versionを無期限にenableのまま放置する。
- シークレットに過剰に広いIAMを付与する。
- 1プロジェクトで異なる環境のシークレットを混在させる。

## クイックチェックリスト
- レプリケーション（auto vs user-managed）を選ぶ。
- reader/adminに最小権限IAMを設定する。
- オーナー/環境のlabelsを追加する。
- ローテーション間隔と自動化を計画する。
- 監査ログ監視を有効化する。
