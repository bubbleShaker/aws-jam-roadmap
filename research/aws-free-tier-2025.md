# AWS 新 Free Tier（2025-07-15〜）調査メモ

Refs #5 #4

> 本ドキュメントには実アカウントID・鍵を一切記載しない（public repo 方針）。

## きっかけ
Jam 学習用に新規 AWS アカウントを作成したところ、**最初から Paid プラン（従量課金）**になり、新 Free Tier の $200 クレジットも付かなかった。原因を調査した。

## 結論（要点）
- 新 Free Tier（Freeプラン＋最大 $200 クレジット）は **「新規顧客限定」**。
- **過去に AWS アカウントを持っていた／持っている人は対象外**。別メール・別エイリアスで新規作成しても、既存顧客と判定されると Paid プランになりクレジットも付かない。
- AWS が「どの識別子（電話番号・クレカ・本人確認情報等）で重複判定するか」は**公式に非公開**。結果として、同一人物の2個目以降のアカウントは新規扱いされない。

公式FAQ の該当記述:
> "You would be ineligible for free plan or Free Tier credits if you have an existing AWS account or have had one in the past."

## 新旧モデル比較

| 観点 | 旧モデル（〜2025-07-14 作成） | 新モデル（2025-07-15〜 作成・新規顧客） | 新規作成でも既存顧客の場合 |
|---|---|---|---|
| 12ヶ月 Free Tier | あり | 廃止 | 無し |
| Always Free 枠 | あり | 新モデルに統合 | （Paid 従量課金） |
| サインアップ時クレジット | 無し | 最大 **$200**（初回$100＋活動で+$100） | **無し** |
| プラン選択 | 無し | Free / Paid を選択 | 実質 Paid 固定 |
| 超過時の挙動 | 従量課金 | Freeプランは**6ヶ月経過 or クレジット消尽の早い方**で課金サービスが一時停止／Paid昇格を促す / Paidは課金 | 従量課金（自動停止ガード無し） |
| Freeプラン期間 | — | **6ヶ月** またはクレジット消尽まで | — |

## 自動的に Paid 扱い／クレジット失効になる条件（FAQより）
- 過去または現在 AWS アカウントを保有している（＝既存顧客）
- アカウントが AWS Organization に参加、または Control Tower landing zone を構築
- AWS Partner Network 参加、Professional Services 契約、Enterprise Agreement、Skill Builder Team 購読、HIPAA/SEC 指定 など

## Jam 学習ロードマップへの含意
1. **本プロジェクトのアカウントは最初から従量課金**。Free Tier の緩衝も $200 クレジットも自動停止ガードも無い前提で計画する。
2. それでも**隔離（本番アカウントと課金・リソースを分離）という目的は達成**できている。既存アカウントを使っても同じく従量課金なので、隔離が効く新アカウントの方が明確に有利。
3. 自動停止ガードが無いぶん、**コスト規律が最重要**になる:
   - 課金が続くリソース（時間課金の NAT GW / Multi-AZ RDS / Aurora / ElastiCache / ALB、データ量従量課金の GuardDuty 等）は**セッション内で必ず削除し残高を目視**
   - **Budgets アラート**（例: 1,000円 / 3,000円）。ただし「止めない通知・最大1日遅れ」である点を忘れない
   - リージョンは `ap-northeast-1` に統一し、消し忘れの飛び地を作らない
4. 予算「月数千円まで」は、上記規律を守れば十分回せる範囲。

## 出典
- [AWS Free Tier FAQs](https://aws.amazon.com/free/free-tier-faqs/)
- [AWS Free Tier update: up to $200 in credits (AWS Blog)](https://aws.amazon.com/blogs/aws/aws-free-tier-update-new-customers-can-get-started-and-explore-aws-with-up-to-200-in-credits/)
- [Confirming eligibility to use AWS Free Tier (AWS Billing docs)](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/free-tier-eligibility.html)
- [Explore AWS services with AWS Free Tier (AWS Billing docs)](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/free-tier.html)
- [re:Post: existing free tier account and $200 eligibility](https://repost.aws/questions/QU1nxMnJfBQXWbcAPvMla_sg/existing-free-tier-account-expiry-on-31-august-2025-is-it-eligible-to-200)
