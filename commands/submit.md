---
description: 公式マーケットプレイスへの全申請フローを実行する（最終送信前に人間確認あり）
---

`skills/claude-plugin-submitter-cmux/SKILL.md` の **Step 1〜10** を順に実行する。SKILL.md は別途読み込まれる前提のため、ここでは実行ポリシーと進捗報告のフォーマットのみを規定する。

## 守るべき掟（最初に読む）

1. **最終 submit ボタンの自動押下は禁止**: 「レビューに提出」は必ず人間が承認した後にクリックする。AI 単独で押してはいけない。
2. **入力後は全フィールドを `cmux browser get value` で再読み出し**して人間に提示する。`screenshot` は最終手段。
3. **Submitter email は最終送信直前に必ず再確認**する。ページ遷移で Claude.ai ログインアカウントの値に戻ることがある。希望値があれば再入力。
4. **Introduction の規約 checkbox は Step 5 で必ず click**。これを忘れると最終 submit で "You must acknowledge the terms to continue." で戻される。
5. **ref はページ遷移ごとに再発番**: 各ページ操作前に必ず snapshot を取り直す。古い ref を使い回さない。

## 事前チェック（Step 0）

以下が満たされない場合は実行を停止し、不足を報告する。

- 環境変数 `CMUX_SOCKET_PATH` が存在する（cmux セッション内で動作）
- `using-cmux` プラグインが利用可能（`cmux browser` コマンドが動く）
- ブラウザ surface を新規に起動できる
- cwd に `plugin.json` または `.claude-plugin/plugin.json` がある
- Claude.ai にログイン済みのブラウザセッションがある（不明な場合は人間に確認）

その後、`/claude-plugin-submitter-cmux:check` 相当のセルフチェックを最初に流す。✗ があれば停止して修正を促す。⚠ のみなら人間に続行可否を確認。

## 実行手順（SKILL.md と対応）

各 Step 完了時に **1 行の進捗ログ** をユーザーに見せる。フォーマット例:

```
[Step 5/10] Introduction: 規約 checkbox 同意 → 次へ ✓
```

- **Step 1**: `plugin.json` 読み取り → 抽出値を要約 1 行で表示
- **Step 2**: 送信前セルフチェック（事前チェックで実施済みならスキップ）
- **Step 3**: 不足情報の対話補完（Examples・Docs URL・Privacy URL 等）
- **Step 4**: 新 surface で `claude.ai/settings/plugins/submit` を開く
- **Step 5**: Introduction 規約 checkbox を click → `is checked` で 1 を確認 → 「次へ」
- **Step 6**: Plugin information 5 フィールドを fill → `get value` で全件検証 → 「次へ」
- **Step 7**: Submission details を fill（Supported platforms = Claude Code、License、Privacy URL、Submitter email）
- **Step 8**: **最終確認** — 全フィールドを `get value` で取得しテキスト形式で人間に提示。Submitter email も再確認。承認を待つ（必須・スキップ禁止）
- **Step 9**: 人間が明示承認した場合のみ「レビューに提出」を click
- **Step 10**: 完了画面の snapshot に「プラグインをレビューに送信しました」が含まれるか検証

## Step 8 の最終確認フォーマット

```
## 送信内容の最終確認

### Plugin information
- Repository: <url>
- Docs URL: <url>
- Name: <name>
- Description: <60 字でトリム>
- Examples: <120 字でトリム>

### Submission details
- Supported platforms: Claude Code ✓ / Claude Cowork <✓ or ☐>
- License: <license>
- Privacy policy URL: <url or "(空)">
- Submitter email: <email>   ← 希望値と一致しているか必ず確認

この内容で「レビューに提出」を押してよいですか？ (yes / no / 修正したい項目を指示)
```

## エラーハンドリング

エラーが発生した場合は SKILL.md の「よくあるミス」「DOM が取れない場合」「ref に関する重要な注意」セクションに従って対処する。代表的な復旧:

- Step 10 で完了 heading が出ない → 多くは規約 checkbox 未同意。Step 5 からやり直す
- ref が見つからない → snapshot を取り直す
- `cmux browser eval` が失敗 → `get value` / `get attr` / `get html` に切り替え

復旧不能なエラーは状況をユーザーに報告して停止する。勝手に再試行ループに入らない。
