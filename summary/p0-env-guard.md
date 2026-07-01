# Phase 0 概要 ― 環境・課金ガード整備（完了）

Issue #4 / Refs #1。安心して壊して学べる AWS 環境を用意し、課金・秘密情報の事故を防ぐ土台を作った。

> ⚠️ この repo は public。アクセスキー・アカウントID・SSO start URL・メール・`.pem` は**絶対に書かない**（実値は `<...>` プレースホルダ）。

## 達成したこと
- **アカウント隔離**: Jam 学習専用の新規 AWS アカウントで本番と課金・リソースを分離（判断は Issue #4 コメント / `knowledge/00-env-setup.md`）。
- **ルート封印 + MFA**: ルートユーザーに MFA。以後ルートは使わず、IAM Identity Center の管理者ユーザー（MFA 済み）で作業。
- **SSO 短期クレデンシャル運用**: IAM Identity Center を東京で有効化し、グループ `jam-admins` × Permission set `AdministratorAccess` を割当。CLI は名前付きプロファイル **`jam`**（`aws configure sso` / region=`ap-northeast-1`）。長期アクセスキーを作らない PLAN の理想形を採用。
- **CLI 疎通確認**: `aws sts get-caller-identity --profile jam` が assumed-role（SSO）で成功。
- **課金ガード**: Cost Explorer 有効化（データ反映は最大24h）。Budgets を2本作成。
  - `jam-warn-1000`（$7・早期警告） / `jam-danger-3000`（$20・危険ライン）
  - 各予算に 実績85% / 実績100% / 予測100% の通知＋メール。予測通知で「超える前」に検知。
- **秘密情報ガード（gitleaks）**: pre-commit フック + `.gitleaks.toml`（PR #11）。今回 **Windows（メイン開発機）にも gitleaks を導入**し、WSL と両環境で fail-closed ガードが効く状態に。

## 重要な決定・気付き
- **請求通貨は USD**。円建て目標（1,000/3,000円）を ≈150円/$ で換算し安全側（低め）に設定。
- **profile 名は必ず `jam` に統一**（`aws configure sso` の既定は長い名前。誤爆防止の運用が命名に依存）。
- **sso-session 名を変えると SSO トークンキャッシュが無効化**され再ログインが必要。
- **gitleaks は環境ごとに導入**（WSL=linux_x64 / Windows=windows_x64、同一版に揃える）。

## 残課題・次フェーズへの申し送り
- Cost Explorer の実データは翌日以降に確認（有効化自体は完了）。
- follow-up 候補: GitHub Actions で gitleaks を回す **CI 二重化**（Issue #12）。ローカルフック未設定でも PR で止める保険。
- 次は **Phase 1（IAM・請求・CLI の地力 / Issue 未起票）**。

## 成果物
- `knowledge/00-env-setup.md` … 手順とハマりどころ（本体）
- 本ファイル `summary/p0-env-guard.md`
