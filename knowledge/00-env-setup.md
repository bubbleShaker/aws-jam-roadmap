# 00 環境・課金ガード整備（Phase 0）手順とハマりどころ

> ⚠️ このファイルは **public repo**。アクセスキー・アカウントID・SSO start URL・`.pem` などは**絶対に書かない**。実値は `<...>` のプレースホルダにする。

## 前提（このプロジェクトの環境）
- AWS CLI v2 導入済み。
- 既存の `default` プロファイルは別プロジェクト用（学習では触らない）。
- 学習用は新規 AWS アカウント＋名前付きプロファイル `jam`（region=`ap-northeast-1`）で隔離する方針（Issue #4 コメント参照）。
- 開発機は Windows。スマホ SSH 接続先は WSL（`~/.aws` は Windows と WSL で別物）。

## 済んだこと
### ルート／管理者 MFA
- ✅ 設定完了済み（2026-07-01 時点）。

### 本番系 SSO プロファイルの一時無効化（Issue #7）
**目的**: Jam 学習中に、業務/本番系の SSO プロファイル（特定の接頭辞で始まる prod / stg の2系統）を誤って叩かないためのガード。

> ℹ️ public repo のため、実プロファイル名は伏せて `<prod-profile>` / `<stg-profile>` / `<prod-session>` / `<stg-session>` と表記する。実値は各自の `~/.aws/config` を参照。

**やったこと**（Windows `~/.aws/config` のみ。WSL 側には元々この本番系プロファイル無し）:
1. config をタイムスタンプ付きでバックアップ
   ```bash
   cp ~/.aws/config ~/.aws/config.bak.$(date +%Y%m%d-%H%M%S)
   ```
2. セクション名に `disabled-` 接頭辞を付与（設定本体は残す＝リネームのみ）
   - `[profile <prod-profile>]`  → `[profile disabled-<prod-profile>]`
   - `[profile <stg-profile>]`   → `[profile disabled-<stg-profile>]`
   - `[sso-session <prod-session>]` → `[sso-session disabled-<prod-session>]`
   - `[sso-session <stg-session>]`  → `[sso-session disabled-<stg-session>]`
   - ✅ 事前に config 全体を確認し、これら2 sso-session を参照する profile は上記2つのみで完結（参照切れの孤児 profile は無し）。
3. 確認
   ```bash
   aws configure list-profiles
   # → disabled-<prod-profile> 等が並び、無効化前の名前が一覧から消えていること
   aws --profile <prod-profile> sts get-caller-identity
   # → "The config profile (<prod-profile>) could not be found" になれば成功
   ```

**復元方法**（本番系をまた使いたくなったら）:
- 各セクション名から `disabled-` を外すだけ（profile 本体の `sso_session = ...` 参照はそのまま残っているので、接頭辞を外せば即整合する）。
- もしくはバックアップ `~/.aws/config.bak.<timestamp>` で丸ごと戻す。

**ハマりどころ**:
- WSL からの SSH 作業では WSL 側 `~/.aws/config` が使われる。今回この本番系は Windows 側だけだったので WSL は対応不要だったが、**環境ごとに `~/.aws` が独立している**点に注意。

## 残タスク（Phase 0 / Issue #4）
- [ ] 作業用 IAM 管理者ユーザー作成（以後ルート不使用）
- [ ] CLI 名前付きプロファイル `jam` を新設（`aws configure --profile jam`, region=`ap-northeast-1`）
- [ ] Budgets 月額アラート（1,000円 / 3,000円）＋ Cost Explorer 有効化
- [ ] gitleaks / git-secrets 導入（コミット前スキャン習慣）
- [ ] CLI 疎通（`aws sts get-caller-identity --profile jam`）
- [ ] 従量課金前提の理解（`research/aws-free-tier-2025.md`）
