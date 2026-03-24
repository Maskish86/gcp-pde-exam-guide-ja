# Cloud SQL

Cloud SQL は、MySQL、PostgreSQL、SQL Server 向けのGCPマネージド リレーショナルデータベースサービスである。プロビジョニング、パッチ適用、バックアップ、高可用性はGCPが担い、スキーマ/ユーザー/クエリは利用者が管理する。

## ユースケース
- パイプラインを支えるメタデータストアや小規模リレーショナルアプリ。
- 分析向けCDCのソース（[[Ingestion/Datastream|Datastream]] 経由）。
- ETLジョブや特徴量パイプライン向けの参照/ルックアップテーブル。

## メンタルモデル
- DBインフラはGoogleが管理し、DBオブジェクトとアクセスは利用者が管理する。
- ストレージは永続で、コンピュートはインスタンス単位の垂直スケール。
- 高可用性は primary + standby と自動フェイルオーバーで実現する。

## コア概念

| 概念                 | 説明                                         |
| -------------------- | -------------------------------------------- |
| Instance             | DBサーバ（リージョン + マシンタイプ）         |
| Database/user        | 論理データベースと認証情報                    |
| Connectivity         | Private IP（VPC）またはPublic IP             |
| Authorized networks  | Public IP接続のためのIP許可リスト             |
| Flags                | エンジン固有の設定                            |

## 接続とセキュリティ

**Private IP:**
- 安定した内部ネットワーク（VPC/ピアリング）に最適。
- パブリック露出とIP許可リストを回避できる。

**Public IP:**
- クライアントがVPC外/インターネット側にいる場合に使う。
- 強固な認証とセキュアな通信が必要。

**Auth Proxy / Connectors:**
- 動的IPクライアントや本番ワークロードで推奨。
- アプリがローカルのproxyへ接続 → proxyがサービスアカウント/Application Default Credentialsで認証 → proxyがCloud SQLへTLS暗号化トンネルを確立する。
- TLSを自動で扱う（手動SSL証明書管理が不要、パブリックIP露出も不要）。
- 短命のOAuth2トークンを使うため、Authorized networksは空にできる。
- 本番で `0.0.0.0/0` の許可リストは避ける。

> IAM（`roles/cloudsql.client`）は「誰が接続できるか」を制御するが、安全なチャネル自体は確立しない。Auth Proxyが認証フロー、トークン交換、TLSを担い、IAMとproxyは組み合わせて使う。

## バックアップと復旧
- 自動バックアップとポイントインタイムリカバリ。
- 復元手順をテストし、保持期間をコンプライアンス要件に揃える。

## 性能とスケーリング
- マシンタイプとストレージを適正サイジングする。
- 読み取りが重いワークロードには read replicas を使う。
- 接続枯渇を避けるため、コネクションプーリングを使う。

## よくある落とし穴
- 動的クライアントにpublic IPの許可リストを使う — IPが変わり、許可リストが静かに破綻する。代わりに Auth Proxy または Cloud SQL Connector を使う。
- HA standby を read replica と混同する — standbyはフェイルオーバー専用で、読み取りは提供しない。読み取りスケールには read replica を追加する。
- サーバレス起因の接続枯渇 — Cloud Run/Functions はコールドスタートで大量接続を生成しうる。コネクションプーリング（PostgreSQLはPgBouncer、またはAuth Proxyのプール設定）を使う。
- ストレージ自動増加は不可逆 — ディスクは自動的に増えるが縮められない。過大なインスタンスは新規作成してデータ移行が必要。
- バックアップ復元テストを省く — 自動バックアップは動いていても、PITR経路や復元手順は障害時まで検証されないことが多い。
- CDC向けDatastream前提条件の不足 — MySQLはbinary logging有効化が必要、PostgreSQLは `wal_level=logical` が必要。これを忘れるとDatastream設定が進まない。

## 連携
- [[Ingestion/Datastream|Datastream]]: Cloud SQLからの分析向けCDC。
- [[Processing/Dataflow|Dataflow]]: ETLとエンリッチメント。
- [[Security/IAM|IAM]]: プロキシ/コネクタ向けIAMロール。

## クイックチェックリスト
- Private IP / Public IP のどちらで接続するか決める。
- 動的IPクライアントには Auth Proxy/Connector を使う。
- 最小権限IAMとDBユーザーを付与する。
- バックアップを構成し、復元テストを行う。
- 監視とアラートを有効化する。
