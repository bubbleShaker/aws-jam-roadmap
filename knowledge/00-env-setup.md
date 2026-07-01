# 00 環境・課金ガード整備（Phase 0）手順とハマりどころ

> ⚠️ このファイルは **public repo**。アクセスキー・アカウントID・SSO start URL・`.pem` などは**絶対に書かない**。実値は `<...>` のプレースホルダにする。

## 前提（このプロジェクトの環境）
- AWS CLI v2 導入済み。
- 既存の `default` プロファイルは別プロジェクト用（学習では触らない）。
- 学習用は新規 AWS アカウント＋名前付きプロファイル `jam`（region=`ap-northeast-1`）で隔離する方針（Issue #4 コメント参照）。
- 開発機は Windows。スマホ SSH 接続先は WSL（`~/.aws` は Windows と WSL で別物）。

## 済んだこと
### ルート／管理者 MFA
- ✅ ルートユーザー MFA 設定完了（2026-07-01 時点）。以後ルートは封印し、下記 SSO 管理者で作業する。
- ✅ 作業用管理者（IAM Identity Center のユーザー）にも MFA 登録済み。

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

### gitleaks を Windows（メイン開発機）でも動かす（Issue #4）
**背景**: フックは fail-closed（gitleaks 不在ならコミット中断）なので、gitleaks が入っていない環境ではそもそもコミットできない。当初は WSL に linux バイナリを入れたが、**メイン開発機は Windows** のため Windows 側にも同バージョンを入れる必要があった（`command -v gitleaks` が Windows の git-bash で解決できないとフックが常に中断する）。

**やったこと**（Windows / git-bash）:
```bash
mkdir -p ~/.local/bin
curl -sSL -o /tmp/gl.zip \
  https://github.com/gitleaks/gitleaks/releases/download/v8.30.1/gitleaks_8.30.1_windows_x64.zip
unzip -o /tmp/gl.zip gitleaks.exe -d ~/.local/bin
~/.local/bin/gitleaks.exe version   # 8.30.1（WSL と同一版に揃える）
```

**ハマりどころ**:
- フックの探索は `command -v gitleaks` → `$HOME/.local/bin/gitleaks` の順。Windows の実体は `gitleaks.exe` なので、**`~/.local/bin` を PATH に通して `command -v` で解決させる**のが確実（`-x "$HOME/.local/bin/gitleaks"` は拡張子違いでヒットしない）。
- WSL 版（linux_x64）と Windows 版（windows_x64）は別バイナリ。**環境ごとに入れる**必要がある（`~/.aws` と同じく環境独立）。
- バージョンは WSL と Windows で揃える（ルール解釈のズレを防ぐ）。

### IAM Identity Center（SSO）＋ `jam` プロファイル（Issue #4）
**方針**: 長期アクセスキーを作らず、**IAM Identity Center の短期クレデンシャル**で学習する（PLAN.md の理想形。public repo が隣にある本プロジェクトでは、ディスクに長期秘密鍵を置かないのが最も安全）。

**コンソール設定の流れ**:
1. **IAM Identity Center を東京リージョンで有効化**（AWS Organizations が自動作成され、このアカウントが管理アカウントになる。単一アカウント運用なので問題なし・無料）。リージョンは後から変えづらいので東京固定。
2. **ユーザー作成**＋そのユーザーに **MFA 登録**（実ユーザー名/メールは伏せる = `<sso-user>`）。
3. **グループ `jam-admins` 作成**し、ユーザーを所属させる。
4. **Permission set = 事前定義済み `AdministratorAccess`**（セッション期間1時間 / リレーステート空欄）。
   - グループに権限を割り当てるのが AWS 推奨の型（ユーザー個人に直接付けない）。Jam の IAM 課題もこの構造前提で出るので学習価値が高い。
5. **AWS アカウントに「グループ `jam-admins` × `AdministratorAccess`」を割当**。

**CLI 設定（`aws configure sso`）**:
- `SSO session name` = `jam` / `SSO start URL` = `<sso-start-url>` / `SSO region` = `ap-northeast-1` / scopes はデフォルト。
- ブラウザ認可後、account 選択 → role = `AdministratorAccess` → region = `ap-northeast-1` → output = `json`。
- ⚠️ **CLI profile name は既定の長い名前を必ず `jam` に上書きする**（以後 `--profile jam` で本番誤爆を防ぐ運用のため）。

**ハマりどころ**:
- profile 名を後からリネームする場合、`~/.aws/config` の3箇所（`[profile <name>]` / `sso_session = <name>` / `[sso-session <name>]`）を揃えて直す。バックアップを取ってから `sed -i 's/<old>/jam/g' ~/.aws/config`。
- **sso-session 名を変えると SSO トークンキャッシュが旧名に紐づいたまま**になり `Error loading SSO Token: Token for jam does not exist` が出る。→ `aws sso login --profile jam` で再ログインすれば解消。
- セッションは1時間で切れる。切れたら `aws sso login --profile jam` で入れ直す（短期クレデンシャルの狙い通り）。

**疎通確認**:
```bash
aws sts get-caller-identity --profile jam
# → arn:aws:sts::<account-id>:assumed-role/AWSReservedSSO_AdministratorAccess_.../<sso-user>
#   assumed-role 経由になっていれば SSO 短期クレデンシャルが機能している
```

### Budgets 月額アラート＋Cost Explorer（Issue #4）
**Cost Explorer**: Billing コンソール → Cost Explorer → 有効化。**データ反映は最大24h**（「24時間後に確認」と出るのは正常＝有効化は成功している）。Budgets は Cost Explorer のデータ反映を待たずに独立して作れる。

**Budgets（2本 / 最初の2本は無料枠）**:
- このアカウントの**請求通貨は USD**。PLAN の「1,000円 / 3,000円」を為替 ≈150円/$ で換算し、安全側（低め）に設定:
  - `jam-warn-1000` … amount **$7**（早期警告）
  - `jam-danger-3000` … amount **$20**（危険ライン）
- 各予算に通知3段階（**実績85% / 実績100% / 予測100%**）＋メール通知先を設定。
  - **予測（Forecasted）通知**を入れる理由: Budgets の反映は最大1日遅れるので、実績が超える「前」に気付くため。自動停止ガードが無いこのアカウントでは早期検知が命綱。

**CLI での検証**（作成後に確認できる）:
```bash
ACCT=$(aws sts get-caller-identity --profile jam --query Account --output text)
aws budgets describe-budgets --account-id "$ACCT" --profile jam \
  --query 'Budgets[].{Name:BudgetName,Limit:BudgetLimit,Type:BudgetType}'
aws budgets describe-notifications-for-budget --account-id "$ACCT" \
  --budget-name jam-danger-3000 --profile jam
```
> ⚠️ `describe-budgets` の出力には account-id・メールが含まれる。**repo には貼らない**（`<account-id>` プレースホルダにする）。

## Phase 0 完了チェック（Issue #4）
- [x] ルート & 管理者（SSO ユーザー）ともに MFA
- [x] 作業用管理者を IAM Identity Center で運用（以後ルート封印）
- [x] CLI 名前付きプロファイル `jam`（SSO / region=`ap-northeast-1`）
- [x] Budgets 月額アラート（$7 / $20）＋ Cost Explorer 有効化
- [x] gitleaks 導入（WSL + Windows 両方）＋ コミット前スキャン習慣
- [x] CLI 疎通（`aws sts get-caller-identity --profile jam`）
- [x] リージョンを `ap-northeast-1` に統一
- [x] 従量課金前提の理解（`research/aws-free-tier-2025.md`）
