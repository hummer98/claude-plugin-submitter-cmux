# claude-plugin-submitter-cmux 開発ガイド

Anthropic 公式 Claude Code plugin マーケットプレイスへの申請を `cmux browser` で自動化するスキル。

## ファイル構成

| ファイル | 役割 |
|---------|------|
| `skills/claude-plugin-submitter-cmux/SKILL.md` | メインスキル定義（AI が読む） |
| `commands/check.md` | `/claude-plugin-submitter-cmux:check` 事前検証コマンド |
| `commands/submit.md` | `/claude-plugin-submitter-cmux:submit` 申請フローコマンド |
| `.claude-plugin/plugin.json` | Plugin マニフェスト |
| `.claude-plugin/marketplace.json` | Marketplace カタログ |
| `README.md` / `README.ja.md` | 人間向けガイド（英・日） |
| `PRIVACY.md` | プライバシーポリシー |
| `CLAUDE.md` | この開発ガイド |
| `LICENSE` | MIT ライセンス |
| `.gitignore` | Git 除外設定 |

## 言語ルール

- **ドキュメント・コメント**: 日本語
- **コード（変数名・関数名）**: 英語
- **README**: 英・日 両言語

## 設計原則

### スコープ

- 対象は **Anthropic 公式マーケットプレイス申請のみ** (`claude.ai/settings/plugins/submit`)
- 他 marketplace は対象外（混乱を避けるため）

### 入力データソース

- 第一: `plugin.json`（name / description / repository / author / license / homepage）
- 第二: README から description 補完
- 第三: 対話でユーザーに確認

### 自動化の境界

- 入力までは自動。**最終送信ボタンは必ず人間確認後**にクリック
- 「レビューに提出」を押す前に snapshot で全フィールドを最終提示

## using-cmux との関係

このスキルは `cmux browser` の操作パターン（snapshot, fill, click, wait）に依存する。それらの一次定義は `using-cmux/SKILL.md` にある。本スキルでは重複定義せず、そちらを参照する形で記述する。

ただし、本スキル固有のフォーム自動化のクセ（後述）はここに集約する。

## フォーム特有のクセ

### Introduction の規約 checkbox

`/settings/plugins/submit` を開いた直後の Introduction ページに同意 checkbox がある。これにチェックを入れずに次のページへ進んでも、最終 submit 時に "You must acknowledge the terms to continue." エラーで Introduction に戻される。**最初のページで必ずチェックを入れること。**

### ref はページ遷移ごとに再発番

`cmux browser snapshot --interactive` の `[ref=eN]` は、ページ移動・DOM 更新のたびに番号が再発行される。各ページで操作する直前に必ず snapshot を取り直して最新 ref を使う。

### Submitter email がページ遷移でリセット

Submission details ページの Submitter email は、ページを行き来すると Claude.ai ログインアカウントの email に戻ることがある。希望のメールアドレスがある場合は **「レビューに提出」を押す直前** に再確認・再入力すること。

### Supported platforms = OS ではなく「Claude Code / Claude Cowork」

UI 上の "Supported platforms" は OS の選択肢ではなく、プラグインが動作する Claude のサーフェス（Claude Code / Claude Cowork）の選択。一般的な Claude Code plugin であれば Claude Code のみチェック。

### snapshot で取れない情報

placeholder のないフィールドや、checkbox のラベルテキストは snapshot だけでは判別できないことがある。その場合は:
- `cmux browser get html <ref>` で実 HTML を取得
- `cmux browser screenshot` は最終手段（トークン消費が大きい）

## メンテナンス手順

1. claude.ai のフォーム構造変更を定期的に確認（半年に一度程度）
2. placeholder 文字列・heading 文字列が変わったら SKILL.md のセレクタ参照を更新
3. 新しいフォームページが追加された場合はクセセクションに追記

## 配布

Plugin として配布。`/plugin marketplace add hummer98/claude-plugin-submitter-cmux` から install。

リリース手順は using-cmux と同じ:
1. `plugin.json` と `marketplace.json` の version を bump
2. コミット
3. tag を打って push (`git tag vX.Y.Z && git push --tags`)
