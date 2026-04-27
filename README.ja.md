日本語 | **[English](README.md)**

# claude-plugin-submitter-cmux

Claude Code plugin の **公式マーケットプレイス申請** (`https://claude.ai/settings/plugins/submit`) を `cmux browser` で自動化する Claude Code スキル。

## モチベーション

公式マーケットプレイスへの申請は claude.ai 上の複数ページからなるフォーム (Introduction → Plugin information → Submission details) への入力が必要。フォームの値の多くは `plugin.json` / README / LICENSE から推測でき、Claude Code スキルが値を準備して `cmux browser` でフォームを埋め、最終送信の直前で人間に確認を求めるフローを再利用可能な形にしたのがこのプラグイン。

## 含まれるもの

| カテゴリ | 説明 |
|---------|------|
| **送信前セルフチェック** | `plugin.json` 必須フィールド、LICENSE / README / PRIVACY.md の存在、リポジトリの到達性を検証 |
| **メタデータ抽出** | `plugin.json` (name, description, repository, author, license) を読み、フォーム値の候補を提示 |
| **フォーム自動化** | `cmux browser` で Introduction → Plugin information → Submission details を辿り、フィールドを埋め、**最終送信の直前で停止**して人間に確認を求める |
| **既知のクセに対応** | snapshot のたびに `[ref=eN]` が再発番される / Submitter email がページ遷移でリセットされる / Introduction の規約 checkbox に同意しないと先に進めない、等を考慮 |

## 前提条件

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) がインストール済み
- [cmux](https://cmux.dev) がインストール済みで、cmux セッション内で Claude Code を実行
- [`using-cmux`](https://github.com/hummer98/using-cmux) プラグインがインストール済み（`cmux browser` 操作パターンに依存）
- cmux のブラウザで Claude.ai にログイン済み
- macOS / Linux

## インストール

```
/plugin marketplace add hummer98/claude-plugin-submitter-cmux
/plugin install claude-plugin-submitter-cmux
```

更新:

```
/plugin update claude-plugin-submitter-cmux
/reload-plugins
```

## 使い方

申請したい plugin リポジトリを cwd にした Claude Code セッションから:

```
/claude-plugin-submitter-cmux:check       # 送信前セルフチェックのみ
/claude-plugin-submitter-cmux:submit      # 全フロー: check → 入力 → 人間確認 → 送信
```

スキルは以下を実行:
1. cwd から `plugin.json` / README / LICENSE / PRIVACY.md を読む
2. 送信前セルフチェックを実行し結果を報告
3. 新しい cmux ブラウザ surface で申請ページを開く
4. `plugin.json` の値でフィールドを埋める。不足分はユーザーに対話で確認
5. 入力済みフォームを表示し、人間にレビューを求める
6. 承認されたら **レビューに提出** をクリック
7. 完了画面が表示されたか検証

## 制限事項

- Anthropic 公式マーケットプレイス (`claude.ai/settings/plugins/submit`) のみ対応
- claude.ai 側のフォーム構造は予告なく変わる可能性がある。セレクタは placeholder ベースで壊れにくくしているが、変更時には更新が必要
- このプラグインは PRIVACY.md や LICENSE を**生成しない**。不足は事前チェックで警告する

## プライバシー

このプラグインは独自にネットワーク通信を行わない。ユーザーが既に認証済みのブラウザセッションを操作するのみ。詳細は [PRIVACY.md](PRIVACY.md)。

## ライセンス

[MIT](LICENSE)
