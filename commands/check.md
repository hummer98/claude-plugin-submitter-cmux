---
description: 申請対象プラグインの送信前セルフチェックのみを実行する（フォーム送信は行わない）
---

`skills/claude-plugin-submitter-cmux/SKILL.md` の **Step 2: 送信前セルフチェック** のみを実行する。フォームは絶対に開かない。

## 実行ポリシー

1. **フォームを開かない**: ブラウザを起動しない。`cmux browser open` 等は実行禁止。
2. **cwd を申請対象とみなす**: 現在の作業ディレクトリにあるプラグインを検査する。
3. **読み取り専用**: ファイルの修正・追加は行わず、不足だけ指摘する。

## 手順

1. cwd に `.claude-plugin/plugin.json`（優先）または `plugin.json` があるか確認。なければ「申請対象のプラグインリポジトリで実行してください」と警告して停止。
2. SKILL.md の Step 2 のチェック項目を順に検証:
   - `name` (必須・kebab-case 推奨)
   - `description` (必須・10 文字以上)
   - `version` (必須・semver)
   - `repository` (必須・GitHub URL)
   - `LICENSE` (存在)
   - `README.md` (存在)
   - `PRIVACY.md` (存在)
3. 結果を下記テンプレートで提示。
4. すべて pass の場合のみ最後に submit コマンドを案内。

## 出力テンプレート（このまま使う）

```
## Pre-submission self-check

| 項目 | 結果 | 詳細 |
|------|------|------|
| plugin.json (name)        | ✓ / ⚠ / ✗ | 値 or 不足理由 |
| plugin.json (description) | ✓ / ⚠ / ✗ | 値（先頭 60 字）or 不足理由 |
| plugin.json (version)     | ✓ / ⚠ / ✗ | 値 or 不足理由 |
| plugin.json (repository)  | ✓ / ⚠ / ✗ | URL or 不足理由 |
| LICENSE                   | ✓ / ⚠ / ✗ | パス or 不足 |
| README.md                 | ✓ / ⚠ / ✗ | パス or 不足 |
| PRIVACY.md                | ✓ / ⚠ / ✗ | パス or 不足 |

凡例: ✓ OK / ⚠ 警告（提出可だが推奨は埋める）/ ✗ エラー（提出前に修正必須）

### 修正のヒント
- ✗ または ⚠ の項目について、具体的な fix を 1 行ずつ列挙
  例: "PRIVACY.md が無い → リポジトリ直下に作成。テンプレは README にリンクあり"
  例: "repository が無い → `plugin.json` に `\"repository\": \"https://github.com/<owner>/<repo>\"` を追記"

### 総合判定
- 🟢 すべて ✓ → 提出可能
- 🟡 ⚠ あり → 提出可能だが改善推奨
- 🔴 ✗ あり → 修正必須
```

## 終了時の案内

- 🟢 の場合のみ最後に下記を 1 行表示:
  > `/claude-plugin-submitter-cmux:submit` で全フローを開始できます
- 🟡 / 🔴 の場合は案内せず、修正依頼で締める。
