# DLP

Cloud Data Loss Prevention（DLP）は、機微データの発見・分類・匿名化（de-identification）を支援する。PII/金融データ/シークレットなどのパターンを検出し、マスキング、トークン化、redactionでプライバシー要件に対応できる。

## ユースケース
- 分析前のrawデータで機微フィールドを特定する。
- 幅広い共有/コンシューマ分析のためにデータを匿名化する。
- 有用性を失わずにプライバシー要件を満たす。
- データレイクのポリシー違反を監視する。

## メンタルモデル
- DLPはデータを検査し、infoTypesに基づく検出結果（findings）を返す。
- 匿名化（de-identification）変換は、共有前に適用する。
- 保存時暗号化（CMEK）は匿名化の代替ではない。

## コア概念
- InfoType：機微パターン検出器（email、SSN、クレジットカードなど）。
- Inspection：検出結果を得るためのスキャン。
- De-identification：機微データの変換（mask、tokenize、redact）。
- Job：ストレージ/BigQueryに対するスケジュール/トリガー型スキャン。

## よく使う匿名化オプション
- マスキング（部分/全体のredaction）。
- トークン化（フォーマット保持トークン）。
- 準識別子（quasi-identifiers）向けのバケッティング/一般化。

## 典型的なプライバシーパイプライン
- 制限データを [[Cloud-Storage|Cloud Storage]] から読む。
- [[Processing/Dataflow|Dataflow]] でDLPを使い、機微フィールドをマスク/トークン化する。
- コンシューマ分析向けに、匿名化データを [[Storage/BigQuery|BigQuery]] に書き込む。

## 連携
- [[Cloud-Storage|Cloud Storage]]：制限バケット内ファイルのスキャンと匿名化。
- [[Processing/Dataflow|Dataflow]]：DLP変換を含むストリーミング/バッチパイプライン。
- [[Storage/BigQuery|BigQuery]]：テーブル検査と匿名化データセット作成。

## よくある落とし穴
- 暗号化をプライバシーと誤解する（DLPは匿名化のため）。
- マスクしすぎて分析の有用性を失う。
- 偽陽性対策のチューニングなしに汎用検出器を使う。

## クイックチェックリスト
- 必要なinfoTypesと検査ルールを定義する。
- フィールドごとに匿名化方式を選ぶ。
- [[Processing/Dataflow|Dataflow]] で処理し、キュレート済みデータセットへ書く。
- サンプル出力で有用性とプライバシーを検証する。
- DLP findings を監視し、ルールを調整する。
