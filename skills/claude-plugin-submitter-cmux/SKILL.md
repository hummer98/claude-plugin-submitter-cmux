---
name: claude-plugin-submitter-cmux
description: Submit a Claude Code plugin to the official Anthropic marketplace via cmux browser automation. Triggers when the user asks to "submit my plugin", "publish to marketplace", or invokes /claude-plugin-submitter-cmux:submit / :check. Reads plugin.json from the cwd, runs pre-submission self-checks, drives the claude.ai submission form (Introduction → Plugin information → Submission details), and stops before final submit for human confirmation.
---

# claude-plugin-submitter-cmux

Anthropic 公式 Claude Code plugin marketplace への申請を `cmux browser` 経由で自動化するスキル。

## いつ起動するか

以下のいずれかでトリガー:
- `/claude-plugin-submitter-cmux:submit` または `/claude-plugin-submitter-cmux:check`
- 「マーケットプレイスに申請したい」「公式に提出したい」「submit my plugin」等の発話
- ユーザーが `claude.ai/settings/plugins/submit` を開いて入力支援を求めた

## 前提条件

- 現在 cmux セッション内で動作している（`CMUX_SOCKET_PATH` が存在）
- `using-cmux` プラグインがインストール済み
- cwd が申請対象のプラグインリポジトリ
- `plugin.json` が `.claude-plugin/plugin.json` または `plugin.json` に存在
- ブラウザの Claude.ai セッションがログイン済み

## 全体フロー

```
1. cwd から plugin.json を読む
2. 送信前セルフチェック (presence + format)
3. 不足情報を対話で補完
4. 新 surface で claude.ai/settings/plugins/submit を開く
5. Introduction で規約 checkbox に同意 → 次へ
6. Plugin information を埋める → 次へ
7. Submission details を埋める
8. 全フィールドを human に提示して確認
9. 承認後「レビューに提出」をクリック
10. 完了画面の確認
```

## Step 1: plugin.json の読み取り

```bash
# .claude-plugin/plugin.json を優先、なければ plugin.json
PLUGIN_JSON=$([ -f .claude-plugin/plugin.json ] && echo .claude-plugin/plugin.json || echo plugin.json)
NAME=$(jq -r '.name' "$PLUGIN_JSON")
VERSION=$(jq -r '.version // "0.1.0"' "$PLUGIN_JSON")
DESCRIPTION=$(jq -r '.description // ""' "$PLUGIN_JSON")
REPOSITORY=$(jq -r '.repository // ""' "$PLUGIN_JSON")
LICENSE=$(jq -r '.license // ""' "$PLUGIN_JSON")
HOMEPAGE=$(jq -r '.homepage // ""' "$PLUGIN_JSON")
AUTHOR=$(jq -r '.author // ""' "$PLUGIN_JSON")
```

`repository` がオブジェクト形式 (`{"type":"git","url":"..."}`) の場合は `.repository.url` を試す。
`homepage` は optional。空でも警告だけにとどめる（フォーム上 Docs URL のフォールバック候補として扱える）。

## Step 2: 送信前セルフチェック

| 項目 | チェック | 失敗時の挙動 |
|------|---------|------------|
| `plugin.json` の `name` | 必須・**kebab-case のみ** (`^[a-z][a-z0-9-]*$`) | エラー表示・停止 |
| `plugin.json` の `description` | 必須・**30〜500 文字目安** | <30: warning / >500: warning / 空: 停止 |
| `plugin.json` の `version` | 必須・semver | warning |
| `plugin.json` の `repository` | 必須・GitHub URL (`https://github.com/...`) | エラー表示・停止 |
| `plugin.json` の `homepage` | 任意。あれば URL 形式 | warning |
| `LICENSE` ファイル | 存在 | warning |
| `README.md` | 存在 | エラー表示・停止 |
| `PRIVACY.md` | 存在 | warning（フォーム入力時に Privacy URL を空にできる） |

> **公式マーケットプレイス特有の制約（推測値あり）**:
> - Plugin name は claude.ai 側で kebab-case のみ許可される可能性が高い。`my_plugin` や `MyPlugin` はバリデーションで弾かれる前提でローカル側でも止める。
> - Description の文字数 30〜500 は経験則（claude.ai 側の正確な上限は未公開）。短すぎる / 長すぎる場合は warning にとどめ、ユーザーに調整を促す。

`/claude-plugin-submitter-cmux:check` はここまで実行して結果を出力するだけ。

## Step 3: 不足情報の対話補完

不足項目があればユーザーに聞く。最低限必要なのは:
- Plugin name (form の "Plugin details" の input)
- Description (form の "Description" textarea)
- Examples (form の "Examples" textarea — 詳細は Step 6 の「Examples の抽出方法」)
- Repository URL
- Documentation URL（README.md の raw URL を提案、なければ `homepage` をフォールバック）
- License (plugin.json から)
- Privacy policy URL (PRIVACY.md の raw URL を提案、なければ空)
- Submitter email（Claude.ai のログインアカウントが自動入力されるが確認）

## Step 4: ブラウザで申請ページを開く

`using-cmux/SKILL.md` の「ブラウザ自動化」セクション参照。

```bash
BSURF=$(cmux browser open https://claude.ai/settings/plugins/submit | awk '{print $2}' | sed 's/surface=//')
cmux browser --surface $BSURF wait --load-state complete --timeout 15
```

## Step 5: Introduction ページ — 規約同意

⚠️ **重要**: Introduction の規約 checkbox に同意しないと、最終 submit 時に "You must acknowledge the terms to continue." で Introduction に戻される。最初のページで必ずチェックを入れる。

```bash
cmux browser --surface $BSURF snapshot --selector "main" --max-depth 100
# → checkbox の ref を探す。heading "Introduction" の近くにある checkbox 行
cmux browser --surface $BSURF click <checkbox-ref>
cmux browser --surface $BSURF is checked <checkbox-ref>   # 1 を確認
cmux browser --surface $BSURF click <次へ-button-ref>
```

### エラー回復

**`is checked` が 0 を返した場合**:
1. checkbox 自体ではなく label をクリックしていた可能性 → snapshot を取り直し、role が `checkbox` の行の ref を再取得
2. それでも 0 なら `cmux browser get html <ref>` で要素種別を確認
3. label を含む親要素の ref を click してみる
4. 3 回失敗したらユーザーに「ブラウザ画面で手動でチェックしてもらえますか」と依頼（このとき `cmux browser screenshot` で画面を提示）

## Step 6: Plugin information ページ

5 つの入力フィールド:

| フィールド | placeholder で識別 | 値の出所 |
|-----------|-------------------|---------|
| GitHub repo URL | `https://github.com/your-org/your-plugin` | `plugin.json` の `repository` |
| Docs URL | `https://your-plugin-docs.com` | README.md の raw URL |
| Plugin name (input) | (placeholder なし、`<input>`) | `plugin.json` の `name` |
| Description (textarea) | `Describe what your plugin does...` | `plugin.json` の `description` + README の冒頭 |
| Examples (textarea) | `Example 1: ...\nExample 2: ...` | README からユースケースを 3 つ |

### placeholder から ref を引くヘルパーパターン

`cmux browser snapshot --interactive` の出力は概ね次の形（簡略化）:

```
- textbox "GitHub repository URL" [ref=e42] placeholder="https://github.com/your-org/your-plugin"
- textbox "Documentation URL"     [ref=e43] placeholder="https://your-plugin-docs.com"
- textbox "Plugin name"           [ref=e44]
- textbox "Description"           [ref=e45] placeholder="Describe what your plugin does..."
- textbox "Examples"              [ref=e46] placeholder="Example 1: ..."
```

placeholder を含む行から ref を抽出する典型パターン:

```bash
SNAP=$(cmux browser --surface $BSURF snapshot --selector "main" --max-depth 100)

# placeholder 文字列でマッチさせて ref を引く
ref_repo=$(echo "$SNAP" | grep -E 'placeholder=".*github\.com/your-org' | grep -oE 'ref=e[0-9]+' | head -1 | cut -d= -f2)
ref_docs=$(echo "$SNAP" | grep -E 'placeholder=".*your-plugin-docs' | grep -oE 'ref=e[0-9]+' | head -1 | cut -d= -f2)
ref_desc=$(echo "$SNAP" | grep -E 'placeholder=".*Describe what your plugin' | grep -oE 'ref=e[0-9]+' | head -1 | cut -d= -f2)
ref_examples=$(echo "$SNAP" | grep -E 'placeholder=".*Example 1' | grep -oE 'ref=e[0-9]+' | head -1 | cut -d= -f2)

# Plugin name は placeholder がないので、accessible name "Plugin name" でマッチ
ref_name=$(echo "$SNAP" | grep -E 'textbox "Plugin name"' | grep -oE 'ref=e[0-9]+' | head -1 | cut -d= -f2)
```

> snapshot の実際の出力フォーマットは `using-cmux/SKILL.md` の「snapshot 出力例」を一次情報源とする。上記は擬似コードであり、フォーマットが変わったら適宜 `grep` のパターンを調整する。

### Examples の抽出方法

README の `## Usage` / `## 使い方` セクションから 3 つのユースケースを抽出する。

```bash
# README から Usage セクションを切り出し（次の H2 まで）
USAGE=$(awk '/^## (Usage|使い方)/{flag=1; next} /^## /{flag=0} flag' README.md)

# 行頭の `-` `*` `/command` などを箇条書きとみなす
# 上位 3 件を Example 1〜3 にマッピング
```

抽出に失敗したり、3 件に満たない場合のフォールバック:
1. `plugin.json` の `description` をベースに Example 1 を生成
2. ユーザーに「Examples に入れる代表的な使い方を 3 つ教えてください」と対話で確認
3. 最低 1 件あれば送信は通る想定（claude.ai 側のバリデーション挙動は未確定 — warning にとどめる）

### 入力と検証

```bash
cmux browser --surface $BSURF fill e$ref_repo  "$REPOSITORY"
cmux browser --surface $BSURF fill e$ref_docs  "$DOCS_URL"
cmux browser --surface $BSURF fill e$ref_name  "$NAME"
cmux browser --surface $BSURF fill e$ref_desc  "$DESCRIPTION"
cmux browser --surface $BSURF fill e$ref_examples "$EXAMPLES"

# 各フィールドを get value で検証
for r in $ref_repo $ref_docs $ref_name $ref_desc $ref_examples; do
  v=$(cmux browser --surface $BSURF get value e$r)
  echo "ref=$r value=${v:0:60}"
done

cmux browser --surface $BSURF click <次へ-button-ref>
```

### エラー回復

**`get value` が空を返した場合**:
1. snapshot を取り直し、ref が変わっていないか確認（DOM 再構築で番号が変わることがある）
2. `cmux browser get html e$ref` で要素種別を確認 — `<input>` ではなく `<textarea>` だった、もしくはラッパー `<div>` を掴んでいたケースあり
3. 別 ref（同じ placeholder を持つ別要素）が存在する場合は `head -1` ではなく `tail -1` で取り直す
4. それでも空なら `cmux browser type e$ref "$VALUE"` を試す（fill ではなく type でキー入力をシミュレート）
5. 3 回失敗したらユーザーに該当フィールドを手動入力するよう依頼

## Step 7: Submission details ページ

| フィールド | 識別 | 値 |
|-----------|------|-----|
| Supported platforms (Claude Code) | checkbox 1 つ目 | ✓ |
| Supported platforms (Claude Cowork) | checkbox 2 つ目 | 通常 ☐（ターミナル系プラグインは Claude Code のみ） |
| License type | placeholder `MIT, Apache 2.0, proprietary, etc.` | `plugin.json` の `license` |
| Privacy policy URL | placeholder `https://your-company.com/privacy` | PRIVACY.md の raw URL（任意） |
| Submitter email | placeholder `contact@your-company.com` | Claude.ai ログインアカウントが自動入力される |

> "Supported platforms" は OS 選択ではなく、プラグインが動作する Claude のサーフェス（Claude Code / Claude Cowork）。一般的な Claude Code plugin であれば Claude Code のみチェックする。

ref 取得は Step 6 と同じ placeholder ベースのパターンを使う。

```bash
SNAP=$(cmux browser --surface $BSURF snapshot --selector "main" --max-depth 100)
ref_license=$(echo "$SNAP" | grep -E 'placeholder=".*MIT, Apache' | grep -oE 'ref=e[0-9]+' | head -1 | cut -d= -f2)
ref_privacy=$(echo "$SNAP" | grep -E 'placeholder=".*your-company\.com/privacy' | grep -oE 'ref=e[0-9]+' | head -1 | cut -d= -f2)
ref_email=$(echo "$SNAP" | grep -E 'placeholder=".*contact@your-company' | grep -oE 'ref=e[0-9]+' | head -1 | cut -d= -f2)
```

### Submitter email リセット対策

⚠️ **Submitter email はページを行き来するとログインアカウントの値に戻ることがある**。希望のメールアドレスがある場合は、Step 8 の最終確認の直前に必ず再確認・再入力する。

```bash
# Step 7 末尾 〜 Step 8 の間に必ず実行
current_email=$(cmux browser --surface $BSURF get value e$ref_email)
if [ "$current_email" != "$DESIRED_EMAIL" ]; then
  cmux browser --surface $BSURF fill e$ref_email "$DESIRED_EMAIL"
  # 再検証
  cmux browser --surface $BSURF get value e$ref_email
fi
```

## Step 8: 最終確認（必須）

`get value` で全フィールドの値を取得し、以下の Markdown テンプレートでユーザーに提示する（**screenshot は最終手段**。snapshot + get value で十分）。

````markdown
## 申請内容の最終確認

### Plugin information
- **Repository**: <REPOSITORY>
- **Docs URL**: <DOCS_URL>
- **Plugin name**: <NAME>
- **Description** (<LEN> 文字):
  > <DESCRIPTION の先頭 200 文字、それ以上は ...>
- **Examples**:
  > <EXAMPLES の先頭 200 文字、それ以上は ...>

### Submission details
- **Supported platforms**: Claude Code <✓ or ☐> / Claude Cowork <✓ or ☐>
- **License**: <LICENSE>
- **Privacy policy URL**: <PRIVACY_URL or "(空)">
- **Submitter email**: <SUBMITTER_EMAIL>

---

上記の内容で「レビューに提出」を押してよろしいですか？ (yes / no / 修正したい項目)
````

ユーザーが「修正したい」と言った場合は、該当ステップに戻って fill し直す。

## Step 9: 提出

ユーザーが承認したら:

```bash
# 直前にもう一度 Submitter email を確認（リセット対策）
cmux browser --surface $BSURF get value e$ref_email

cmux browser --surface $BSURF click <レビューに提出-button-ref>
sleep 2
cmux browser --surface $BSURF snapshot --selector "main" --max-depth 80
```

### エラー回復

**snapshot の結果に "Introduction" heading や規約 checkbox が再び含まれている場合** = 規約未同意としてリトライさせられた状態:
1. ユーザーに「規約 checkbox の同意状態がリセットされたため再同意します」と通知
2. Step 5 を実行（規約 checkbox を click）
3. 各ページの値が保持されているか snapshot で確認 — 消えていたら Step 6/7 を再実行
4. Step 9 を再試行

**snapshot に "You must acknowledge the terms" の文字列が見える場合** も同様に Step 5 からやり直す。

**snapshot に他のバリデーションエラー（"required field" 等）が見える場合**:
1. エラー文言の near にある field の ref を取得
2. ユーザーにエラー内容を提示し、当該フィールドを再入力
3. Step 9 を再試行

## Step 10: 完了の検証

snapshot に `プラグインをレビューに送信しました` (または英語版 `Plugin submitted for review`) の heading が含まれていれば成功。

含まれていなければ Step 9 のエラー回復に従う。3 回連続で失敗したら自動化を諦め、`cmux browser screenshot` を撮ってユーザーに状態を提示し、手動操作を依頼する。

## ref に関する重要な注意

`cmux browser snapshot --interactive` の `[ref=eN]` は **ページ遷移・DOM 更新ごとに再発番される**。各ページに移動した直後に必ず snapshot を取り直して最新 ref を使うこと。

```bash
# ❌ 間違い — 古い ref を使い回す
cmux browser --surface $BSURF click e123   # 別ページの ref かもしれない

# ✅ 正しい — ページごとに snapshot を取り直す
cmux browser --surface $BSURF click <next-button-ref>
sleep 1
cmux browser --surface $BSURF snapshot --selector "main" --max-depth 100   # 新 ref
cmux browser --surface $BSURF fill <new-ref> "..."
```

## DOM が取れない場合

`cmux browser eval` は WKWebView の制約で動かないケースがある。代替手段:

| 取得したい情報 | 代替 |
|--------------|------|
| 要素の値 | `cmux browser get value <ref>` |
| 属性 | `cmux browser get attr <ref> <name>` |
| HTML | `cmux browser get html <ref>` |
| チェック状態 | `cmux browser is checked <ref>` |
| placeholder のないフィールドの正体 | `get html <ref>` で `<input>` か `<textarea>` か確認 |

## よくあるミス

| ミス | 正しい方法 |
|------|-----------|
| Introduction の規約同意を飛ばす | 必ず checkbox を click し `is checked` で 1 を確認してから先へ進む |
| 古い ref を使い回す | ページ遷移後は必ず snapshot を取り直す |
| Submitter email を確認せず submit | 最終送信前に `get value` で再確認、必要なら再入力 |
| 視覚確認のため毎回 screenshot を撮る | `snapshot` + `get value` でほぼ全て取得可能。screenshot はトークン消費が大きいので最終手段 |
| eval で DOM 取得を試みる | WKWebView では失敗する。`get value` / `get attr` / `get html` を使う |
| `--surface` で他ワークスペースを操作 | `using-cmux` の `cmux-send` 等を経由する（自動解決） |
| Plugin name に snake_case や CamelCase を使う | kebab-case のみ。事前チェックでローカル側でも止める |

## using-cmux への参照

cmux browser の基本操作（snapshot, fill, click, wait, get value 等）は `using-cmux/SKILL.md` の「ブラウザ自動化」セクションが一次情報源。本スキルでは申請フォーム固有の操作手順とエラー回復のみを記述する。
