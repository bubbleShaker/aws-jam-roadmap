# AWS Jam 調査メモ

Refs #2

## AWS Jam とは
- AWS の**実シナリオ課題**をチームで解きながら学ぶゲーミフィケーション型ハンズオンイベント。
- 「聞く」ではなく「実際に手を動かす」体験型学習プラットフォーム。
- AWS Summit Japan ではブースで実施。**当日枠は争奪戦**（参加報告だと午後の枠が早期に埋まる／要こまめなチェック）。

## 競技形式
- 1チームで複数の課題を**制限時間内**に解く（課題数はイベントにより変動。数問〜十数問規模が一般的）。
- 課題は1問あたり**1時間以上**かかることもある実践的シナリオ。
- 各課題には難易度があり、難しいほど**高得点**。
- **リアルタイムのリーダーボード**で他チームと順位を競う。
- 個人参加・チーム参加・既存チーム合流など柔軟。

## 採点メカニズム（ここが攻略の肝）
- 課題ごとに配点。難易度が高いほど点が大きい。
- **ヒント/クルー（clue）を使うと減点**される。
  - → **自力で速く正解するほど高得点**。地力勝負。
- ベストプラクティス実装を**自動検証**する仕組みがある（CloudFormation で各課題用の AWS 環境が新規アカウントに展開される）。

## 出題ドメイン（200以上の課題プールから）
- **Security**: IAM/アクセス制御、データ保護・暗号化、ネットワーク/インフラセキュリティ、フォレンジック、インシデント対応
- **Networking**: VPC、ルーティング、接続性のトラブルシュート
- **Serverless**: Lambda/API Gateway/DynamoDB、サーバーレスのトラブルシュート
- **DevOps / 自動化**: CI/CD、IaC、remediation at scale（大規模修復）、自動化
- **IaC**: CloudFormation（YAML テンプレート）
- **AI/ML**, **Analytics**, **アプリ現代化**, **IoT**, **Compliance**

## 勝つために効く力（逆算）
1. **トラブルシューティング即応力** — 「動かない」状態から原因を素早く切り分ける。VPC/IAM/ルーティング/権限まわりが頻出。
2. **IaC で素早く環境を組む力** — CloudFormation を読める・直せる。
3. **横断的なサービス知識** — 浅くても広く。どのサービスのどの設定が怪しいか当たりをつけられる。
4. **AI なしで解く速度** — 本番は生成 AI 禁止。マネジメントコンソール/CLI を手に馴染ませる。
5. **チーム連携** — 課題を分担し、得意ドメインを持つ。

## 本番「生成 AI 禁止」の含意
- 普段 AI に頼って解いていると本番で手が止まる。
- → 学習段階で**意図的に AI を切って解く演習**を組み込む必要がある（特にトラブルシュート系）。
- バイブコーディングは「教材・採点スクリプト・復習問題・IaC 雛形の生成」など**準備側**に限定して活用する。

## 出典
- [AWS Jam 公式](https://jam.awsevents.com/)
- [Jam Getting Started Guide (PDF)](https://aws-jam-docs.s3-us-west-2.amazonaws.com/aws-jam-getting-started.pdf)
- [Jam User Guide (overview)](https://nshun.github.io/aws-jam-user-docs/jam-overview/)
- [Gamify your cloud journey (AWS Blog)](https://aws.amazon.com/blogs/training-and-certification/gamify-your-cloud-journey-how-aws-jams-are-created-and-why-they-matter/)
- [AWS Jam Journeys: role-based challenges (AWS Blog)](https://aws.amazon.com/blogs/training-and-certification/role-based-jam-journeys/)
- [Immersive learning / AWS Jam](https://aws.amazon.com/training/digital/aws-jam/)
- [AWS Summit Japan 2026](https://aws.amazon.com/jp/events/summits/japan/)
