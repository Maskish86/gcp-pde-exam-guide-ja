# Cloud KMS

Cloud Key Management Service（Cloud KMS）は、GCPおよびアプリケーションのデータ暗号化に使う暗号鍵を管理する。鍵ライフサイクルの集中管理、IAMベースのアクセス制御、暗号操作の監査ログを提供する。

## ユースケース
- [[Storage/BigQuery|BigQuery]] や [[Cloud-Storage|Cloud-Storage]] などでCMEKを有効化する。
- 鍵のローテーション/無効化/破棄ポリシーを制御する。
- 暗号化の職務とデータアクセスを分離する（セキュリティ + コンプライアンス）。
- カスタムアプリケーション/パイプラインでデータを暗号化する。

## メンタルモデル
- KMSは **envelope encryption** を使う：サービスがDEKでデータを暗号化し、KMSはKEK（key-encryption-key）でDEKを保護する。
- 付与するのは「鍵を **使う** 権限」であり、データを読む権限とは別物。
- key versionは不変。ローテーションで新しいversionが作られ、旧versionは既存データの復号のために残る。
- 鍵をdisableすると新規のencrypt/decryptが止まる。破棄は不可逆のため、まずdisableする。

## コア概念

| 概念 | 説明 |
| --- | --- |
| **Key ring** | 鍵のリージョン単位の論理コンテナ |
| **Crypto key** | 1つ以上のversionを持つ名前付き鍵 |
| **Key version** | 実際の鍵マテリアル（enabled / disabled / destroyed） |
| **Key purpose** | 対称のencrypt/decrypt、または非対称のsign/verify |
| **Software key** | 既定の保護レベル（鍵マテリアルはGoogle基盤内） |
| **HSM key** | ハードウェア保護で耐タンパ性。コンプライアンス要件で必要な場合がある |
| **EKM key** | 鍵マテリアルはon-prem/外部KMSに残り、GCPへ入らない |

## CMEK（Customer-Managed Encryption Keys）

データセット/バケット/ジョブにKMS鍵を紐づけることで、サービスがDEKをあなたの鍵で保護する。

- BigQueryテーブルはGMEK→CMEKをインプレースで切り替えられない。CMEKテーブルを新規作成し、データをコピーする。

**バケットのデフォルトCMEK設定**（以後の書き込みをすべてCMEKに強制）:
```bash
gcloud storage buckets update gs://my-bucket \
  --default-encryption-key=projects/.../cryptoKeys/my-key
```

**移行時の落とし穴（独立した2つの制御）:**

| 制御 | 対象 |
| --- | --- |
| copy/rewrite時の明示キー | その時点で既存オブジェクトを再暗号化する |
| バケットのデフォルトCMEK | 以後の書き込みを自動的にCMEKへ強制する |

デフォルトCMEKを設定しないと、後続アップロードが黙ってGMEK（Google管理暗号化）になることがある。必ず両方を設定する。

**鍵侵害時の対応:**
1. 新しい鍵を作成し、バケット/データセットのデフォルトとして設定する。
2. 既存データを再書き込み/コピーする — ローテーションだけでは保存オブジェクトを再暗号化しない。
3. 再暗号化を確認した後、侵害鍵をdisableし、その後destroyする。

## HSM と Cloud EKM

**Cloud HSM:** タンパー耐性ハードウェアに基づいた鍵。暗号化操作はHSM内で発生し、生の鍵マテリアルは露出しない。コンプライアンス要件がハードウェア保証を必要とする場合に使う。

**Cloud EKM:** 鍵マテリアルはon-prem HSMまたは外部キー管理に残る。GCPサービスはEKMバックのKMS鍵をCMEKとして使う（暗号化自体はGoogle管理だが、鍵はGoogle Cloudに入らない）。

> 要件が「鍵マテリアルをオンプレにのみ保持」であれば、答えはCloud EKM。Cloud KMS/Cloud HSMへ鍵を取り込むと、鍵マテリアルはGCP内に置かれる。クライアントサイド暗号化は、BigQueryの保存時暗号化のようなCMEK制御の代替にはならない。

## ローテーションとライフサイクル
- 各鍵にローテーション間隔と次回ローテーション時刻を設定する。
- 新規データは最新のprimary versionを自動で使う。
- 旧versionは、既存データの復号のために有効なまま残る。
- そのversionで暗号化されたデータが再暗号化済み、または不要であることを確認してからdestroyする。

## アクセス制御とIAM

| ロール | 用途 |
| --- | --- |
| `roles/cloudkms.cryptoKeyEncrypterDecrypter` | パイプラインサービスアカウント（暗号化 + 復号） |
| `roles/cloudkms.cryptoKeyEncrypter` | 書き込み専用パイプライン（暗号化のみ） |
| `roles/cloudkms.admin` | 鍵管理 — 控えめに使い、鍵ユーザーと分離する |

- プロジェクトとキーリング単位でスコープをアクセスする（グローバルではなく）。
- 鍵管理者と鍵ユーザーを分離する — 最小権限の原則。

## 監査とコンプライアンス
- すべての鍵操作（create/use/disable/destroy）はCloud Audit Logsに記録される。
- 復号イベントとポリシー変更を追跡し、コンプライアンス要件を満たす。
- ローテーション期間とログ保持期間をコンプライアンスフレームワークに合わせる。

## よくある落とし穴
- すべての暗号化データの再暗号化前にkey versionをdestroyする — destroyは不可逆で永続的なデータ損失を招く。まずdisableし、そのversionを参照する稼働データがないことを確認してからdestroyする。
- すべてに同じ鍵を使う — 侵害された または 設定ミスの鍵は保護データ全体に影響する。データドメインや機微度レベル別にキーをスコープし、環境別にキーリングを分離する。
- サービスアカウントへ `cryptoKeyEncrypterDecrypter` 付与を忘れる — 操作は黙って権限拒否で失敗する。リソース（BigQuery、GCS）はデータを保持するが、サービスアカウントは鍵自体に明示的なIAMが必要。
- ローテーションは既存データを再暗号化しない — ローテーションは将来の操作向けの新しいprimary versionを作るだけで、既存オブジェクトは旧versionで暗号化されたままになる。明示的に再書き込み/コピーして再暗号化する。
- 操作単位のCMEK設定をしてバケット/データセットのデフォルトがない — 将来の書き込みが黙ってGMEK（Google管理暗号化）にフォールバックしうる。操作単位だけでなく、リソースのデフォルトCMEKを必ず設定する。
- CMEKがプライバシーやPII要件を満たすと仮定する — CMEKは保存時の暗号化を制御するだけで、データコンテンツは制御しない。匿名化やPII マスキングには [[Security/DLP|DLP]] を別途使う。
- KMS鍵と保護リソース間でリージョンが不一致 — クロスリージョン鍵使用はレイテンシを増やし、データレジデンシーポリシー違反になりうる。キーリングリージョンはリソースリージョンと一致させる。グローバル鍵はレジデンシー保証をバイパスする。

## 連携
- [[Cloud-Storage|Cloud Storage]]: すべての新規オブジェクト書き込み向けバケットデフォルトCMEK。
- [[Storage/BigQuery|BigQuery]]: データセット/テーブルCMEKおよびクエリワーカー向けジョブレベルCMEK。
- [[Processing/Dataflow|Dataflow]] / [[Processing/Dataproc|Dataproc]]: ワーカーディスクとパイプラインステージングデータ向けCMEK。
- [[Security/IAM|IAM]]: 鍵使用と管理向けの最小権限ロール。

## クイックチェックリスト
- リージョンを選び、データレジデンシー要件に合わせる。
- 決める：対称 vs 非対称、ソフトウェア vs HSM vs EKM。
- ローテーションポリシーを設定し、復旧/再暗号化ステップを文書化する。
- 最小権限IAMを付与 — 鍵管理者とユーザーを分離する。
- 操作単位のキーではなく、バケット/データセットのデフォルトCMEKを設定する。
- 鍵使用とポリシー変更向けCloud Audit Logの監視を有効化する。
