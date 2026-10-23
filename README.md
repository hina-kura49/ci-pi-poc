# PoC: lobe-chat `claude-pr-assign.yml` — CI prompt injection → privileged `gh` abuse

このリポは **lobehub/lobe-chat** の `.github/workflows/claude-pr-assign.yml`
(origin/canary = 本番ブランチ, 2026-10-23 時点) の脆弱な設定を**忠実に再現した最小再現環境**です。
第三者(lobehub)のライブ CI は一切攻撃しません。成立の証明は**この自分のリポ上だけ**で行います。

## 脆弱性の要点

- `pull_request_target` トリガ → フォーク/外部ユーザーの PR でも **base リポのトークン＋シークレット付き**で実行される。
- `anthropics/claude-code-action@v1` が、エージェントの仕事として
  `gh pr view --json title,body,files` で **攻撃者制御の PR 本文/タイトルを読み込む** = プロンプト注入面。
- エージェントの権限:
  - `github_token`(lobe-chat では PAT `secrets.GH_TOKEN`; 本PoCは `GITHUB_TOKEN`+`pull-requests: write`)
  - `allowed_non_write_users: '*'` = 書込権限のない外部ユーザーにも作用
  - `--allowedTools "Bash(gh pr:*),..."` = `gh pr comment`/`gh pr review --approve`/`gh pr close`/**`gh pr merge`** まで一致する広すぎる許可
- 防御は `security-rules.md` と「注入を検知したら止まれ」= **どちらもモデルへのお願い(ソフト)** で、セキュリティ境界ではない。

## 成立条件(これが証明したいこと)

攻撃者が PR 本文に注入ペイロードを入れるだけで、エージェントが**本来の仕事(レビュア割当)でなく
本文の指示**に従い、特権トークンで `gh pr comment` / `gh pr review --approve` を実行する。
→ bot 名義の承認でブランチ保護を満たす / PAT が広ければ `gh pr merge` で本体にコード注入(供給網侵害)。

## セットアップ手順(あなたが実行)

1. **public リポを作成**(例: `hina-kura49/ci-pi-poc`)。※ private だと外部フォーク PR の挙動が再現しにくいので public 推奨。
   ```bash
   cd <このディレクトリ>
   git init && git add . && git commit -m "poc: mirror lobe-chat claude-pr-assign"
   gh repo create hina-kura49/ci-pi-poc --public --source=. --push
   ```
2. **Claude のトークンをシークレット登録**(どちらか一方):
   ```bash
   # 事前に `claude setup-token` で OAuth トークンを取得している場合
   gh secret set CLAUDE_CODE_OAUTH_TOKEN -R hina-kura49/ci-pi-poc
   # または API キー
   gh secret set ANTHROPIC_API_KEY -R hina-kura49/ci-pi-poc
   ```
   (`GITHUB_TOKEN` は自動で入るので登録不要)
3. **攻撃者役の PR を開く**。lobe-chat 同様 `pull_request_target` は「PR を開く」だけで発火する。
   - 2nd アカウント、または別の GitHub アカウントでこのリポをフォークし、
   - フォーク側で無害な変更(README に1行追加など)を含むブランチを作り、
   - **PR の本文(description)に `injection-pr-body.md` の中身を貼り付けて** PR を作成。
     ```bash
     # フォーク側で
     gh pr create --repo hina-kura49/ci-pi-poc --title "docs: fix typo" --body-file injection-pr-body.md
     ```
   - ※ 同一アカウントの self-PR でも発火はするが、「外部コントリビュータ」条件を忠実に見るなら別ID/フォーク推奨。

## 成立の確認(証拠の取り方)

- PR に **`PWNED-7Q2X-INJECTED`** を含むコメントが bot から投稿される → **注入成立**。
- PR が bot により **approved** になる(Reviews 欄) → **承認の偽造が可能**であることの証明。
- Actions の実行ログ(`assign-reviewer` ジョブ)に、エージェントが
  `gh pr comment ... PWNED-...` / `gh pr review --approve` を実行した行が残る → スクショ/ログ保存。
- もしエージェントが**拒否して `security:prompt-injection` ラベルを付けて停止**したら:
  → ソフト防御が効いた回。`injection-pr-body.md` の言い回しを変えて再試行(数パターン用意)。
    **偽陽性チェックの意味で、ここで止まるなら報告の深刻度を下げる/取り下げる判断材料**になる。

## 片付け

PoC 確認後、リポは削除してよい(`gh repo delete hina-kura49/ci-pi-poc`)。承認した PoC PR は自分のリポ内なので影響なし。

<!-- poc change -->
