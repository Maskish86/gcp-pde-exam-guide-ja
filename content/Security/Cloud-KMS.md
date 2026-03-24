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
1. Create a new key and set it as the bucket/dataset default.
2. Rewrite/copy existing data — rotation alone does not re-encrypt stored objects.
3. 再暗号化を確認した後、侵害鍵をdisableし、その後destroyする。

## HSM と Cloud EKM

**Cloud HSM:** Keys backed by tamper-resistant hardware; cryptographic operations occur inside the HSM, raw key material never exposed. Use when compliance requires hardware assurance.

**Cloud EKM:** 鍵マテリアルはon-prem HSMまたは外部キー管理に残る。GCPサービスはEKMバックのKMS鍵をCMEKとして使う（暗号化自体はGoogle管理だが、鍵はGoogle Cloudに入らない）。

> 要件が「鍵マテリアルをオンプレにのみ保持」であれば、答えはCloud EKM。Cloud KMS/Cloud HSMへ鍵を取り込むと、鍵マテリアルはGCP内に置かれる。クライアントサイド暗号化は、BigQueryの保存時暗号化のようなCMEK制御の代替にはならない。

## ローテーションとライフサイクル
- 各鍵にローテーション間隔と次回ローテーション時刻を設定する。
- 新規データは最新のprimary versionを自動で使う。
- 旧versionは、既存データの復号のために有効なまま残る。
- そのversionで暗号化されたデータが再暗号化済み、または不要であることを確認してからdestroyする。

## アクセス制御とIAM

| Role | Use For |
| --- | --- |
| `roles/cloudkms.cryptoKeyEncrypterDecrypter` | Pipeline service accounts (encrypt + decrypt) |
| `roles/cloudkms.cryptoKeyEncrypter` | Write-only pipelines (encrypt only) |
| `roles/cloudkms.admin` | Key management — use sparingly, separate from key users |

- Scope access by project and key ring, not globally.
- Separate key admins from key users — principle of least privilege.

## 監査とコンプライアンス
- すべての鍵操作（create/use/disable/destroy）はCloud Audit Logsに記録される。
- Track decrypt events and policy changes to satisfy compliance requirements.
- Align rotation periods and log retention with your compliance framework.

## よくある落とし穴
- すべての暗号化データの再暗号化前にkey versionをdestroyする — destroyは不可逆で永続的なデータ損失を招く。まずdisableし、そのversionを参照する稼働データがないことを確認してからdestroyする。
- One key for everything — a compromised or misconfigured key affects all protected data; scope keys by data domain or sensitivity tier, and separate key rings by environment.
- Forgetting to grant `cryptoKeyEncrypterDecrypter` to service accounts — operations fail silently with permission denied; the resource (BigQuery, GCS) holds the data but the service account needs explicit IAM on the key itself.
- Key rotation does not re-encrypt existing data — rotation creates a new primary version for future operations only; existing objects remain encrypted under the old version; explicitly rewrite or copy objects to re-encrypt them.
- Setting per-operation CMEK without a bucket or dataset default — future writes can silently fall back to GMEK (Google-managed encryption); always set the default CMEK on the resource, not just per-operation.
- Assuming CMEK satisfies privacy or PII requirements — CMEK controls encryption at rest, not data content; use [[Security/DLP|DLP]] for de-identification and PII redaction separately.
- Mismatched regions between KMS key and protected resource — cross-region key use adds latency and may violate data residency policy; key ring region must match the resource region; global keys bypass residency guarantees.

## 連携
- [[Cloud-Storage|Cloud Storage]]: bucket default CMEK for all new object writes.
- [[Storage/BigQuery|BigQuery]]: dataset/table CMEK and job-level CMEK for query workers.
- [[Processing/Dataflow|Dataflow]] / [[Processing/Dataproc|Dataproc]]: CMEK for worker disks and pipeline staging data.
- [[Security/IAM|IAM]]: least-privilege roles for key usage and key administration.

## クイックチェックリスト
-  Choose region and align with data residency requirements.
-  Decide: symmetric vs asymmetric, software vs HSM vs EKM.
-  Set rotation policy and document recovery/re-encryption steps.
-  Grant least-privilege IAM — separate key admins from key users.
-  Set bucket/dataset default CMEK, not just per-operation keys.
-  Enable Cloud Audit Log monitoring for key usage and policy changes.
