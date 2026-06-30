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

### gitleaks による秘密情報ガード（Issue #4）
**目的**: public repo に AWS アクセスキー・アカウントID・SSO start URL を**コミットする前**に止める。

**仕組み**:
- `hooks/pre-commit`（repo 管理・実行可能）がステージ済み差分を `gitleaks git --staged` でスキャンし、秘密らしき文字列があればコミットを **fail-closed で中断**する。
- フックは `.git/hooks/`（git 管理外）ではなく repo の `hooks/` に置き、`git config core.hooksPath hooks` で指す。→ フック自体も git 管理でき、他マシンでも再現できる。
- ルールは `.gitleaks.toml`。gitleaks 同梱のデフォルト（AWS アクセスキー検出など）を継承し、**デフォルトでは拾えない「12桁アカウントID（ARN内 / `account_id` ラベル近接）」「awsapps.com の SSO start URL」を custom rule で追加**した（gitleaks 単体ではこの2つは検出されないことを実測確認）。
- 許可リストは同 `.gitleaks.toml`。プレースホルダ `<...>`・AWS 公式ドキュメントの例示キー・ダミー account-id `123456789012` を誤検知しないよう登録済み。認証情報ファイル（`.aws/`・`*.bak`）は**あえてスキャン除外しない**（`git add -f` でバイパスされた時の最後の砦にするため）。

**導入手順（他マシン／WSL 再構築時に再現）**:
1. gitleaks バイナリを入れる（このプロジェクトは v8.30.1 を使用）。
   ```bash
   mkdir -p ~/.local/bin
   curl -sSL -o /tmp/gl.tar.gz \
     https://github.com/gitleaks/gitleaks/releases/download/v8.30.1/gitleaks_8.30.1_linux_x64.tar.gz
   tar -xzf /tmp/gl.tar.gz -C /tmp gitleaks
   install -m 0755 /tmp/gitleaks ~/.local/bin/gitleaks
   gitleaks version   # 8.30.1
   ```
   `~/.local/bin` が PATH に無くてもフックが絶対パスでフォールバックするので動く。
2. フックを有効化（repo ルートで一度だけ）。
   ```bash
   git config core.hooksPath hooks
   ```

**動作確認の型**:
- ニセの AWS シークレットをステージ → `bash hooks/pre-commit` が **exit 非0 で中断**すれば OK。
- プレースホルダ `<prod-profile>` や `123456789012` だけのファイル → **通過**すれば許可リストが効いている。

**ハマりどころ**:
- `core.hooksPath` は**ローカル設定**なので clone/再構築のたびに張り直す（コミットには乗らない）。手順2を README/この doc に残しておく理由がこれ。
- どうしても1回だけ通したい時のみ `git commit --no-verify`。常用は禁止（ガードが死ぬ）。
- gitleaks 未導入の環境ではフックが**わざとコミットを止める**（黙って素通りさせない fail-closed 設計）。

## 残タスク（Phase 0 / Issue #4）
- [ ] 作業用 IAM 管理者ユーザー作成（以後ルート不使用）
- [ ] CLI 名前付きプロファイル `jam` を新設（`aws configure --profile jam`, region=`ap-northeast-1`）
- [ ] Budgets 月額アラート（1,000円 / 3,000円）＋ Cost Explorer 有効化
- [x] gitleaks 導入（コミット前スキャン習慣）← 本セクションで完了
- [ ] CLI 疎通（`aws sts get-caller-identity --profile jam`）
- [x] 従量課金前提の理解（`research/aws-free-tier-2025.md`）
