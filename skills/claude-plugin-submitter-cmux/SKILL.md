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
```

`repository` がオブジェクト形式 (`{"type":"git","url":"..."}`) の場合は `.repository.url` を試す。

## Step 2: 送信前セルフチェック

| 項目 | チェック | 失敗時の挙動 |
|------|---------|------------|
| `plugin.json` の `name` | 必須・kebab-case 推奨 | エラー表示・停止 |
| `plugin.json` の `description` | 必須・10 文字以上 | エラー表示・停止 |
| `plugin.json` の `version` | 必須・semver | warning |
| `plugin.json` の `repository` | 必須・GitHub URL | エラー表示・停止 |
| `LICENSE` ファイル | 存在 | warning |
| `README.md` | 存在 | エラー表示・停止 |
| `PRIVACY.md` | 存在 | warning（フォーム入力時に Privacy URL を空にできる） |

`/claude-plugin-submitter-cmux:check` はここまで実行して結果を出力するだけ。

## Step 3: 不足情報の対話補完

不足項目があればユーザーに聞く。最低限必要なのは:
- Plugin name (form の "Plugin details" の input)
- Description (form の "Description" textarea)
- Examples (form の "Examples" textarea — README から代表的な使い方を3つ抽出するか対話で聞く)
- Repository URL
- Documentation URL（README.md の raw URL を提案）
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

⚠️ **重要**: Introduction の規約 checkbox に同意しないと、最終 submit 時に "You must acknowledge the terms to continue." で戻される。

```bash
cmux browser --surface $BSURF snapshot --selector "main" --max-depth 100
# → checkbox の ref を探す。placeholder 内の文字列 "Anthropic" や heading "Introduction" が目印
cmux browser --surface $BSURF click <checkbox-ref>
cmux browser --surface $BSURF is checked <checkbox-ref>   # 1 を確認
cmux browser --surface $BSURF click <次へ-button-ref>
```

## Step 6: Plugin information ページ

5 つの入力フィールド:

| フィールド | placeholder で識別 | 値の出所 |
|-----------|-------------------|---------|
| GitHub repo URL | `https://github.com/your-org/your-plugin` | `plugin.json` の `repository` |
| Docs URL | `https://your-plugin-docs.com` | README.md の raw URL |
| Plugin name (input) | (placeholder なし、`<input>`) | `plugin.json` の `name` |
| Description (textarea) | `Describe what your plugin does...` | `plugin.json` の `description` + README の冒頭 |
| Examples (textarea) | `Example 1: ...\nExample 2: ...` | README からユースケースを 3 つ |

```bash
# 最新 snapshot から ref を取得
cmux browser --surface $BSURF snapshot --selector "main" --max-depth 100
cmux browser --surface $BSURF fill <e-repo>  "$REPOSITORY"
cmux browser --surface $BSURF fill <e-docs>  "$DOCS_URL"
cmux browser --surface $BSURF fill <e-name>  "$NAME"
cmux browser --surface $BSURF fill <e-desc>  "$DESCRIPTION"
cmux browser --surface $BSURF fill <e-examples> "$EXAMPLES"
# 値を get value で全部検証
cmux browser --surface $BSURF click <次へ-button-ref>
```

## Step 7: Submission details ページ

| フィールド | 識別 | 値 |
|-----------|------|-----|
| Supported platforms (Claude Code) | checkbox 1 つ目 | ✓ |
| Supported platforms (Claude Cowork) | checkbox 2 つ目 | 通常 ☐（ターミナル系プラグインは Claude Code のみ） |
| License type | placeholder `MIT, Apache 2.0, proprietary, etc.` | `plugin.json` の `license` |
| Privacy policy URL | placeholder `https://your-company.com/privacy` | PRIVACY.md の raw URL（任意） |
| Submitter email | placeholder `contact@your-company.com` | Claude.ai ログインアカウントが自動入力される |

⚠️ **Submitter email は ページ遷移で自動入力値に戻ることがある**。「レビューに提出」を押す直前に再確認すること。

## Step 8: 最終確認（必須）

`get value` で全フィールドの値を取得し、ユーザーにテキスト形式で提示する。**screenshot は最終手段** — snapshot + get value で十分。

```
Plugin information:
- Repository: https://github.com/.../...
- Docs: https://github.com/.../README.md
- Name: my-plugin
- Description: ... (truncated)
- Examples: ... (truncated)

Submission details:
- Claude Code: ✓ / Claude Cowork: ☐
- License: MIT
- Privacy URL: https://github.com/.../PRIVACY.md
- Submitter email: user@example.com

「レビューに提出」を押しますか？
```

## Step 9: 提出

ユーザーが承認したら:

```bash
cmux browser --surface $BSURF click <レビューに提出-button-ref>
sleep 2
cmux browser --surface $BSURF snapshot --selector "main" --max-depth 80
# heading "プラグインをレビューに送信しました" が出れば成功
```

## Step 10: 完了の検証

snapshot に `プラグインをレビューに送信しました` が含まれていれば成功。

含まれていなければエラー（典型例: Introduction ページに戻されている → 規約 checkbox 未同意）。原因をユーザーに報告し、必要なら Step 5 からやり直す。

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
| Introduction の規約同意を飛ばす | 必ず checkbox を click してから先へ進む |
| 古い ref を使い回す | ページ遷移後は必ず snapshot を取り直す |
| Submitter email を確認せず submit | 最終送信前に `get value` で再確認 |
| 視覚確認のため毎回 screenshot を撮る | `snapshot` + `get value` でほぼ全て取得可能。screenshot はトークン消費が大きいので最終手段 |
| eval で DOM 取得を試みる | WKWebView では失敗する。`get value` / `get attr` / `get html` を使う |
| `--surface` で他ワークスペースを操作 | `using-cmux` の `cmux-send` 等を経由する（自動解決） |

## using-cmux への参照

cmux browser の基本操作（snapshot, fill, click, wait, get value 等）は `using-cmux/SKILL.md` の「ブラウザ自動化」セクションが一次情報源。本スキルでは申請フォーム固有の操作手順のみを記述する。
