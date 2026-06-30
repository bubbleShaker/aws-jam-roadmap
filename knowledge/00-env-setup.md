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

### 本番系 SSO プロファイルの完全削除（Issue #7 → #9）
**目的**: Jam 学習中に、業務/本番系の SSO プロファイル（特定の接頭辞で始まる prod / stg の2系統）を誤って叩かないためのガード。

> ℹ️ public repo のため、実プロファイル名は伏せて `<prod-profile>` / `<stg-profile>` / `<prod-session>` / `<stg-session>` と表記する。実値は各自の `~/.aws/config` を参照。

#### なぜ「リネーム無効化」から「完全削除」に切り替えたか（Issue #9）
当初（Issue #7）はセクション名に `disabled-` 接頭辞を付けて無効化した。しかし**これは不十分**だった:
- 設定本体は残っているため、新しい名前で `aws --profile disabled-<prod-profile> ...` と打てば**依然として使えてしまう**。
- 「古いコマンド履歴でうっかり叩く」事故は防げるが、「使えなくする」目的は果たせていなかった。

→ そこで Issue #9 で **セクションごと完全削除**に切り替えた。

**やったこと**（Windows `~/.aws/config` のみ。WSL 側には元々この本番系プロファイル無し）:
1. config をタイムスタンプ付きでバックアップ（復元の保険。**public repo には絶対に入れない**ローカル限定ファイル）
   ```bash
   cp ~/.aws/config ~/.aws/config.bak.$(date +%Y%m%d-%H%M%S)
   ```
2. 本番系の `profile` / `sso-session` セクション（`<prod-profile>` / `<stg-profile>` / `<prod-session>` / `<stg-session>`）を**丸ごと削除**（`localstack` / `default` は維持）。
   - 残すのは `[profile localstack]` と `[default]` のみ。
3. 確認
   ```bash
   aws configure list-profiles
   # → localstack と default だけになり、本番系（disabled- 含む）が一切出ないこと
   aws --profile <prod-profile> sts get-caller-identity
   aws --profile disabled-<prod-profile> sts get-caller-identity
   # → どちらも "The config profile (...) could not be found" になれば成功
   ```

**復元方法**（本番系をまた使いたくなったら）:
- ローカルバックアップ `~/.aws/config.bak.<timestamp>` から該当セクションを戻す。
- もしくは `aws configure sso` で再設定する（sso_start_url / account_id / role を再入力）。

**ハマりどころ**:
- WSL からの SSH 作業では WSL 側 `~/.aws/config` が使われる。今回この本番系は Windows 側だけだったので WSL は対応不要だったが、**環境ごとに `~/.aws` が独立している**点に注意。
- 無効化は「リネーム」では不十分。**完全に使えなくしたいならセクションごと削除**する（リネームは新名で呼べてしまう）。

## 残タスク（Phase 0 / Issue #4）
- [ ] 作業用 IAM 管理者ユーザー作成（以後ルート不使用）
- [ ] CLI 名前付きプロファイル `jam` を新設（`aws configure --profile jam`, region=`ap-northeast-1`）
- [ ] Budgets 月額アラート（1,000円 / 3,000円）＋ Cost Explorer 有効化
- [ ] gitleaks / git-secrets 導入（コミット前スキャン習慣）
- [ ] CLI 疎通（`aws sts get-caller-identity --profile jam`）
- [ ] 従量課金前提の理解（`research/aws-free-tier-2025.md`）
