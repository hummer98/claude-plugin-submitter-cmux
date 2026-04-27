---
description: 申請対象プラグインの送信前セルフチェックのみを実行する（フォーム送信は行わない）
---

cwd を申請対象プラグインのリポジトリと見なし、`skills/claude-plugin-submitter-cmux/SKILL.md` の「Step 2: 送信前セルフチェック」のみを実行する。

チェック内容:
- `.claude-plugin/plugin.json`（または `plugin.json`）の必須フィールド
- `LICENSE` / `README.md` / `PRIVACY.md` の存在
- `repository` フィールドの URL 形式

結果をユーザーに表示し、不足や警告があれば指摘する。フォームは開かない。
