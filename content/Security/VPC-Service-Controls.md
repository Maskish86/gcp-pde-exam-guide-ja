# VPC Service Controls

VPC Service Controls（VPC SC）は、Google管理サービスの周囲にサービスペリメータを作り、データ持ち出し（exfiltration）リスクを下げる。IAMが許可していても、プロジェクト/サービス単位でアクセスを制限できる。

## 使う理由
- 機微サービス（例：[[Ingestion/PubSub|Pub/Sub]]、[[Storage/BigQuery|BigQuery]]、[[Cloud-Storage|Cloud-Storage]]）へのプロジェクト間アクセスを防ぐ。
- 意図しない/悪意ある認証情報の使用を含め、定義したペリメータ外からのアクセスを遮断する。
- 機微データに対して、IAMを補完するペリメータ制御を追加する。

## 要点
- VPC SCはVPCネットワークレベルではなく、プロジェクト/サービス（Google API）レベルで適用される。
- 保護対象はGoogle管理サービスであり、VM間トラフィックではない。
- IAMは権限ベースだが、VPC SCは「境界」を追加し、権限があっても拒否できる。
- ペリメータ外のプロジェクトは、明示的に追加されない限り保護サービスへアクセスできない。
- VPC SCはGoogle APIのアクセス制御（ペリメータ）であり、ネットワーク到達性は **提供しない**。パブリックインターネットなしでGCS/BQへ到達すべき内部IP専用ワーカー（例：Dataflow）には、接続性として **Private Google Access** を使う。

## よくある落とし穴
- VPCファイアウォールルールで `pubsub.googleapis.com` アクセスを制限できると誤解する。
- ペリメータ分離をIAMだけに頼る。
- 必要なときに新規プロジェクトをペリメータへ追加し忘れる。
