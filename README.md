# aws-jam-roadmap

AWS Summit 2027 Japan の **AWS Jam で3位以内**を狙うための、1年間ハンズオン学習ロードマップ。

## ゴール
- AWS Jam（実シナリオ課題をチームで時間内に解く競技）で上位入賞できる「地力」をつける
- 本番は**生成 AI 使用禁止**。よって AI なしで速く解けるトラブルシュート力・IaC 構築力を鍛える
- SAA 認定を学習の背骨に据えつつ、毎週ハンズオンで手を動かす

## 学習者プロフィール（前提）
- AWS 習熟度: ほぼ初心者スタート
- 学習時間: 週3〜5時間
- 環境予算: 月数千円まで（学習用アカウントは従量課金。Free Tier・クレジット無し前提でコスト規律を徹底。詳細 `research/aws-free-tier-2025.md`）
- 資格: SAA 中心に組み込む

## バイブコーディングの使い分け
- **準備フェーズ（OK）**: 学習教材・採点スクリプト・復習問題の自動生成、IaC の雛形づくり
- **競技演習（AI 禁止で解く）**: Jam 想定の模擬課題は AI を一切使わず時間制限つきで解く

## ディレクトリ
- `research/` — AWS Jam・各サービスの調査メモ
- `summary/` — 各フェーズ完了時の概要
- `knowledge/` — つまずきポイントと解説
- `PLAN.md` — 1年ロードマップ本体

## 安全運用（public repo / 課金事故防止）
- **秘密情報を絶対にコミットしない**: アクセスキー・`~/.aws/credentials`・`.pem`・アカウントID は厳禁。`.gitignore` で除外済み＋コミット前に gitleaks でスキャンする。
  - clone / 環境再構築のたびに pre-commit フックを有効化する（`core.hooksPath` はコミットに乗らないため）:
    ```bash
    git config core.hooksPath hooks
    ```
    gitleaks 本体の導入手順とハマりどころは `knowledge/00-env-setup.md` を参照。
- **時間課金リソースはセッション内で必ず削除し残高を目視**（NAT GW / Multi-AZ RDS / Aurora / ALB / GuardDuty 等）。Budgets は「止めない通知」に過ぎない。
- 詳細は `PLAN.md` の Phase 0 を参照。

## 進捗管理
GitHub Issue で管理。Epic（ロードマップ全体）配下にフェーズごとの Issue を立てる。
