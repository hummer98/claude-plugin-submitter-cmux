日本語 | **[English](README.md)**

# claude-plugin-submitter-cmux

Claude Code plugin の **Anthropic 公式マーケットプレイス申請** を `cmux browser` で自動化し、最終送信は人間が確認するスキル。

## ステータス

**v0.1.0 — 骨子段階・未検証。** 2026 年初頭時点で観測した `claude.ai/settings/plugins/submit` のフォーム構造と既知のクセ（規約 checkbox、ref 再発番、Submitter email リセット）をスキル化したもの。初回実行は付き添い前提で。Issue・PR 歓迎 — [コントリビューション](#コントリビューション) 参照。

## モチベーション

公式マーケットプレイスへの申請は claude.ai 上の複数ページからなるフォーム (Introduction → Plugin information → Submission details) への入力が必要。フォームの値の多くは `plugin.json` / README / LICENSE から推測でき、Claude Code スキルが値を準備して `cmux browser` でフォームを埋め、最終送信の直前で人間に確認を求めるフローを再利用可能な形にしたのがこのプラグイン。

## 含まれるもの

| カテゴリ | 説明 | 境界 |
|---------|------|------|
| **送信前セルフチェック** | `plugin.json` 必須フィールド、`LICENSE` / `README.md` / `PRIVACY.md` の存在、リポジトリ URL 形式を検証 | 自動 |
| **メタデータ抽出** | `plugin.json` (name, description, repository, author, license, homepage) を読み、フォーム値の候補を提示 | 自動 |
| **フォーム自動化** | `cmux browser` で Introduction → Plugin information → Submission details を辿りフィールドを埋める | 自動 |
| **最終確認** | 入力済みの全フィールド値をチャットに表示してレビューさせる | 自動・人間レビュー |
| **最終送信クリック** | 「レビューに提出」ボタン | **人間が確認後に実行** |
| **Anthropic 側のレビュー** | 提出後の承認・却下・追加質問対応 | **対象外** |

## 自動化されること / 人間に残ること

- **自動**: リポジトリのメタデータ読み取り、フォームを開く、フィールド入力、ページ遷移後の `[ref=eN]` 再取得、全フィールドのレビュー表示
- **人間**: Anthropic 規約のレビュー、入力値が正しいかの判断、最終送信ボタンの押下、提出後に Anthropic から来るやりとりへの返信

## 前提条件

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) がインストール済み
- [cmux](https://cmux.dev) がインストール済みで、cmux セッション内で Claude Code を実行
- [`using-cmux`](https://github.com/hummer98/using-cmux) プラグインがインストール済み（`cmux browser` 操作パターンに依存）
- cmux のブラウザで Claude.ai にログイン済み
- macOS / Linux

> **なぜ Windows 不可?** このプラグインは `cmux browser` を駆動するが、cmux 自体が macOS / Linux 向けのターミナル多重化ツール。上流が Windows 非対応なので、こちらも非対応。

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

**申請したい plugin リポジトリ**を cwd にした、cmux 内の Claude Code セッションから:

```
/claude-plugin-submitter-cmux:check       # 送信前セルフチェックのみ（安全なドライラン）
/claude-plugin-submitter-cmux:submit      # 全フロー: check → 入力 → 人間確認 → 送信
```

リスクなく試したい場合は `:check` を先に実行する。ブラウザを一切開かずフォームにも触らない。`plugin.json` と周辺ファイルを検証して結果を表示するだけ。

`:submit` の全フロー:
1. cwd から `plugin.json` / `README.md` / `LICENSE` / `PRIVACY.md` を読む
2. 送信前セルフチェックを実行し結果を報告
3. 新しい cmux ブラウザ surface で申請ページを開く
4. `plugin.json` の値でフィールドを埋める。不足分はユーザーに対話で確認
5. 入力済みフォームを表示し、人間にレビューを求める
6. 承認されたら **レビューに提出** をクリック
7. 完了画面が表示されたか検証

## トラブルシューティング

| 症状 | 原因 | 対処 |
|------|------|------|
| 「cmux browser not found」でスキルが中断 | cmux 外で実行、または `using-cmux` 未インストール | cmux セッションを起動し `/plugin install using-cmux` |
| 最終送信後に Introduction ページに戻され "You must acknowledge the terms to continue." エラー | Introduction の規約 checkbox 未同意 | `:submit` を再実行（Step 5 で checkbox を click する）。再現する場合は cmux ブラウザ surface で Introduction ページを目視確認 |
| 申請ページではなく Claude.ai ログイン画面が表示される | ブラウザセッションが未認証 | cmux ブラウザで `https://claude.ai` を開いてログインし、`:submit` を再実行 |
| 最終確認時に Submitter email が想定外 | ページ遷移時に Claude.ai ログインアカウントの email にリセットされる | 最終確認のプロンプトで承認前に email を編集 |
| `[ref=eN]` エラー / 「element not found」 | ページ遷移後に古い ref が残っている | スキルはページごとに snapshot を取り直す。それでも失敗するなら再実行 — claude.ai 側の DOM がページ間で変化している可能性 |
| `:check` で `PRIVACY.md missing` 警告が出るが公開したくない | フォームの Privacy URL は任意 | `PRIVACY.md` を追加するか、警告を無視してフォームの Privacy URL は空欄のまま提出 |

## 制限事項

- Anthropic 公式マーケットプレイス (`claude.ai/settings/plugins/submit`) のみ対応。サードパーティ marketplace は明示的に対象外。
- claude.ai 側のフォーム構造は予告なく変わる可能性がある。セレクタは placeholder ベースで壊れにくくしているが、Anthropic が UI を更新したら追従が必要。
- このプラグインは `PRIVACY.md` や `LICENSE` を**生成しない**。事前チェックで不足を警告するが、作成はユーザーの責任。
- v0.1.0 は実際の申請を通したエンドツーエンド検証が未完了。荒削りな部分がある前提で利用してほしい。

## ロードマップ

- 実申請でのエンドツーエンド検証とセレクタ追従
- スクリーンキャスト / アニメーション GIF の追加
- claude.ai が placeholder を変えた際の自動回復
- CI 連携: plugin リポジトリへの push ごとに `:check` を回すフック

## プライバシー

このプラグインは独自にネットワーク通信を行わない。ユーザーが既に認証済みのブラウザセッションを操作するのみ。詳細は [PRIVACY.md](PRIVACY.md)。

## コントリビューション

Issue・Pull Request は [`hummer98/claude-plugin-submitter-cmux`](https://github.com/hummer98/claude-plugin-submitter-cmux/issues) へ。`skills/claude-plugin-submitter-cmux/SKILL.md` に記載のないフォームのクセに当たった場合は、該当ページと観測した placeholder 文字列を添えて報告してもらえると最も助かる。

## ライセンス

[MIT](LICENSE)
