---
description: 公式マーケットプレイスへの全申請フローを実行する（最終送信前に人間確認あり）
---

cwd を申請対象プラグインのリポジトリと見なし、`skills/claude-plugin-submitter-cmux/SKILL.md` の Step 1〜10 を順に実行する。

- Step 1: `plugin.json` 読み取り
- Step 2: 送信前セルフチェック
- Step 3: 不足情報の対話補完
- Step 4: ブラウザ surface で申請ページを開く
- Step 5: Introduction で規約同意 → 次へ
- Step 6: Plugin information 入力 → 次へ
- Step 7: Submission details 入力
- Step 8: **最終確認をユーザーに提示**（必須・スキップしない）
- Step 9: 承認後「レビューに提出」をクリック
- Step 10: 完了画面の検証

最終送信ボタンを押すのはユーザーが明示的に承認した場合のみ。
