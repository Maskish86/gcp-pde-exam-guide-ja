# Looker

Looker は、GCPのエンタープライズBI/セマンティックモデリング基盤である。BigQueryなどのソース上にガバナンスされたデータモデル（LookML）を定義し、信頼できるメトリクスをダッシュボードや埋め込みアプリへ提供する。

## ユースケース
- ビジネスメトリクス/ディメンションの単一の正（single source of truth）を提供する。
- LookMLで定義を中央集約し、メトリクスドリフトを防ぐ。
- 行/列レベルのアクセス制御付きで、ガバナンスされたダッシュボードを提供する。
- 一貫したロジックで、社内アプリに分析を埋め込む。

## メンタルモデル
- LookMLがセマンティック層を定義し、SQLはクエリ時に生成される。
- modelがrawテーブルと、キュレート済みビジネスビューを分離する。
- キャッシュは性能を上げるが、データ鮮度問題を隠すことがある。
- 権限はDBだけでなく、セマンティック層でも強制される。

## コア概念
- LookML：dimensions/measures/joins/explores を定義するモデリング言語。
- Model：explores と connections の集合。
- Explore：エンドユーザーがクエリ可能なビュー。
- View：テーブルまたはderived tableへのマッピング。
- Derived table：事前計算のためのSQL定義テーブル（PDT/DT）。
- Dashboard：explores上に保存された可視化。

## よくあるパターン
- [[Storage/BigQuery|BigQuery]] のキュレート済みテーブル上にLookMLを構築する。
- クエリ時に高コストな重い変換は、derived tablesで事前計算する。
- ユーザー属性に基づいて行レベルセキュリティを強制する。
- LookMLをバージョン管理し、レビュー/ロールバック可能にする。

## 連携
- [[Storage/BigQuery|BigQuery]]: primary warehouse backend.
- [[Cloud-Storage|Cloud Storage]]: data lake staging feeding BigQuery.
- [[Governance/Dataplex|Dataplex]]: governance and metadata alignment.
- [[Security/IAM|IAM]]: access control foundation for connections and users.

## セキュリティとアクセス制御
- ウェアハウス接続は最小権限にする。
- Looker権限を [[Security/IAM|IAM]] グループと整合させる。
- LookMLで行/列セキュリティを適用し、データポリシーを強制する。

## 運用と信頼性
- BigQuery側でダッシュボードのロード時間とクエリコストを監視する。
- キャッシュを戦略的に使い、鮮度期待を文書化する。
- 下流レポート破壊を避けるため、LookML変更はレビューを通す。

## よくある落とし穴
- キュレート済みではなくrawテーブル上でモデリングする。
- explore間でメトリクスロジックを重複させる。
- クエリコストとキャッシュ無効化の挙動を無視する。

## クイックチェックリスト
- LookMLでコアメトリクス/ディメンションを定義する。
- exploreの参照先をキュレート済みBigQueryテーブルにする。
- 行レベルセキュリティルールを設定し、アクセスをテストする。
- LookMLのレビュー/デプロイ手順を整備する。
- オーナーとデータ鮮度期待を文書化する。
