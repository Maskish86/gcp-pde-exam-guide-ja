# Datastream

Datastream は、GCPのマネージド Change Data Capture（CDC）サービスである。ソースデータベースの行レベル変更を取り込み、[[Storage/BigQuery|BigQuery]] や [[Cloud-Storage]] などの宛先へ、最小限の運用負荷でストリーミングする。

## ユースケース
- OLTPデータを分析システムへ低レイテンシでレプリケーションする。
- Debezium やカスタムログリーダーを運用せずにCDCパイプラインを構築する。
- 監査やリプレイのために履歴トレイルを保持する。
- 変換/ルーティングのために、（多くは [[Processing/Dataflow|Dataflow]] の）ストリーミングパイプラインへ供給する。
- Kafka やカスタムCDCクラスタなしで、フルマネージドなサービスを使う。

## メンタルモデル
- Datastream は継続的な変更のために、テーブルスキャンではなくデータベースログを読む。
- テーブル単位に順序付けられた変更イベント（insert/update/delete）を出力する。
- 初期バックフィルは、継続的な変更キャプチャとは別。
- 通常はrawなCDCイベントを着地させ、そこからキュレート済みテーブルへ変換する。

## コア概念
- **Connection profile**: ソース/宛先の認証情報と接続情報。
- **Stream**: ソース、宛先、オプション（オブジェクト、フィルタ、バックフィル）を定義する。
- **Backfill**: 選択テーブルの初期スナップショット。
- **Change stream**: ログ由来の継続的なCDCイベント。
- **Private connectivity**: ソースへのセキュアなネットワーク経路。

## ソースと宛先

### Sources
- MySQL/PostgreSQL の [[Cloud-SQL|Cloud SQL]]。
- 自己管理のMySQL/PostgreSQL（適切な接続が前提）。
- Oracle（CDC対応）。

### Destinations
- [[Storage/BigQuery|BigQuery]]（CDCテーブル + メタデータ）。
- [[Cloud-Storage|Cloud Storage]]（下流処理向けのファイルベースCDC）。

## バックフィルとCDCフロー
典型的なライフサイクル:
1. connection profile を作成する（ソース + 宛先）。
2. stream を定義する（スキーマ/テーブル選択 + バックフィル設定）。
3. バックフィルを実行し、ベースラインのテーブルを取り込む。
4. 継続CDCを開始し、データを最新に保つ。

バックフィルの指針:
- 初期ロードに使い、行数と主キーを検証する。
- ソース負荷に敏感なら、オフピークにスケジュールする。

## データ形状とセマンティクス
Datastream は最終状態ではなく変更を出力する:
- insert/update/delete は、メタデータ（timestamp、source）付きのイベントとして到着する。
- 現在状態テーブルを作るには、変更を適用する必要がある。
- delete は明示的なイベントとして到着する。下流はtombstoneを扱える必要がある。

## 宛先別パターン

### To [[Storage/BigQuery|BigQuery]]
- Datastream がCDCテーブルを書き出し、`MERGE` でキュレート済みテーブルを構築する。
- 順序付けにはCDCメタデータのイベント時刻を使う。

### To [[Cloud-Storage|Cloud Storage]]
- CDCファイルを [[Processing/Dataflow|Dataflow]] またはバッチジョブで処理できる。
- リプレイ可能な追記専用パイプラインに向く。

## 性能とコスト
- 必要なスキーマ/テーブルにstreamを絞り、負荷とコストを下げる。
- ソースと宛先のラグを監視する。
- 大きなバックフィルは高コストになりうるため、先にデータ量を検証する。

## 信頼性と監視
- streamの健全性、ラグ、エラーログを追跡する。
- 欠損を避けるため、ソースのログ保持期間がstreamラグを上回るようにする。
- 定期的な行数やチェックサムで突合する。

## セキュリティとガバナンス
- CDC用のデータベースユーザーは最小権限にする。
 - ソースDB（例：**Oracle**）がVPC内にある場合、パブリックIP露出を避けるためにプライベート接続（PSC/VPC peering）を選ぶ。
- 宛先でCMEKが必要なら、[[Cloud-KMS|Cloud KMS]] 経由で有効化する。

## よくある落とし穴
- 主キーがない（下流でクリーンにupsertしにくい）。
- ソースログの保持期間が不足し、データ欠損が起きる。
- 変更適用せずにCDCテーブルを「最終形」と誤解する。
- ソースと宛先のリージョン不一致。

## クイックチェックリスト
- ソースDBのログ設定と保持期間を確認する。
- 宛先（[[Storage/BigQuery|BigQuery]] vs [[Cloud-Storage|Cloud Storage]]）と下流の変換計画を決める。
- バックフィルのタイミングを計画し、行数を検証する。
- 信頼できるupsertのために主キーの存在を確認する。
- ラグとエラー率の監視を設定する。
