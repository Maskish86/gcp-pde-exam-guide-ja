# Memorystore

Memorystore は、Redis と Memcached のためのGCPマネージド インメモリ データストアである。サブミリ秒のキャッシュとエフェメラルなデータ保管を、サーバ管理なしで提供する。**これはキャッシュであり、system of record ではない** — データはメモリ上にあり、永続ストレージの代替ではない。

## ユースケース
- ホットキーや高コストなクエリ結果をキャッシュし、[[Storage/BigQuery|BigQuery]] やOLTPシステムへの負荷を下げる。
- セッションデータ、feature flags、レート制限カウンタを保持する。
- サブミリ秒の読み取りが必要なサービング層を高速化する。
- キャッシュから参照/ルックアップテーブルを供給し、ストリーミングパイプライン（[[Processing/Dataflow|Dataflow]]）をエンリッチする。

## メンタルモデル
- system of record ではなく、キャッシュ（またはエフェメラルなストア）である。
- データはメモリ上にあり、永続化は限定的で、耐久ストレージの代替ではない。
- 性能は、キー設計、eviction policy、メモリサイジングに依存する。
- アクセスは **private IP（VPC peering のみ）** で行う — パブリックエンドポイントはない。

## コア概念

| 概念            | 説明                                                                                    |
| --------------- | --------------------------------------------------------------------------------------- |
| Instance        | マネージドなRedisまたはMemcachedのデプロイ                                              |
| Tier (Redis)    | Basic = 単一ノード（レプリカなし）。Standard = primary + レプリカ（自動フェイルオーバー） |
| Eviction policy | メモリ枯渇時の挙動（LRU, volatile, noeviction など）                                     |
| Read replicas   | Redis Standard: レプリカを背後に持つread endpoint。primaryから読み取り負荷を逃がす        |
| Failover        | Redis Standard: ゾーン障害時にレプリカを自動昇格。Basic と Memcached: なし               |
| AUTH / TLS      | Redis: パスワード認証と転送時暗号化をサポート。Memcached: どちらもなし                  |

## Redis vs Memcached

| 項目               | Redis                                           | Memcached                       |
| ------------------ | ----------------------------------------------- | ------------------------------- |
| データ構造          | 多様: lists, sets, sorted sets, hashes, streams | 単純なキー/バリューのみ          |
| 永続化              | 任意（RDB snapshots）                           | なし                            |
| HA / フェイルオーバー | Standard tier のみ                              | なし                            |
| Read replicas      | Standard tier のみ                              | なし                            |
| Pub/Sub            | あり                                            | なし                            |
| 適する用途          | 多機能キャッシュ、セッション、leaderboards      | 単純で高スループットなキャッシュ |

- **移行:** **Memorystore for Redis** は **RDB（Redis Database）スナップショット** → GCS → import をサポートする（RDB = Redisデータのポイントインタイムなディスクスナップショット）。**Memcached** には永続化/importがないため、コールドキャッシュとして扱う（アンチパターン: statefulな移行を期待する）。

## パフォーマンスとコスト
- メモリサイジングが主要なコストレバー — ピーク負荷に備えたヘッドルームを持たせてプロビジョニングする。
- eviction rate を監視する。evictions が多いのは、メモリ不足またはキー設計が不適切なサイン。
- Standard tier はレプリカ分のコストが増える。開発/非クリティカル用途はBasicを使う。
- read replicas（Standard）は、読み取りが重いパターンでprimaryの負荷を下げる。
- eviction policy はヒット率に影響する: `allkeys-lru` は任意キーをevict、`volatile-lru` はTTL付きキーのみevict、`noeviction` はメモリ満杯でエラーを返す。

## セキュリティとガバナンス
- アクセスは **VPC peering のみ** — Memorystore にpublic IPはない。
- Redisでは **転送時暗号化（TLS）** を有効化する（Memcachedは未対応）。
- **AUTH**（Redisパスワード）で、VPC内のデータプレーンアクセスを制限する。
- リソース管理（インスタンス作成/削除）には [[Security/IAM|IAM]] を使う。IAMはデータプレーンアクセスを制御しない。

## よくある落とし穴
- 重要データの唯一のコピーとしてMemorystoreを使う。
- メモリを過小プロビジョニングしてevictionsを発生させる。
- リージョン配置とクライアントまでのレイテンシを無視する。
- HAを期待してMemcachedを選ぶ — レプリカもフェイルオーバーもない。
- 無停止が必要な本番ワークロードでBasic tierを使う。

## 連携
- [[Storage/BigQuery|BigQuery]]: 高コストなクエリ結果をキャッシュし、slot消費を減らす。
- [[OperationalDBs/Cloud-SQL|Cloud SQL]] / [[OperationalDBs/Spanner|Spanner]]: DB読み取りをキャッシュして低レイテンシにサーブする。
- [[Processing/Dataflow|Dataflow]]: ストリームエンリッチのため、キャッシュから参照/ルックアップテーブルを供給する。
- Cloud Run / App Engine: セッション保存、レート制限カウンタ。

## クイックチェックリスト
- Redis（多機能） vs Memcached（単純なキー/バリューのみ）を選ぶ。
- Basic（開発/テスト） vs Standard（本番、HA必須）を選ぶ。
- ヘッドルーム込みでメモリをサイジングし、ユースケースに合わせてeviction policyを定義する。
- 本番のRedisはTLSとAUTHを有効化する。
- クライアントと同一リージョンにインスタンスを配置する。
- メモリ使用量、eviction rate、接続数を監視する。
