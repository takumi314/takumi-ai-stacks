---
name: auto-commit-msg
description: |
    Conventional Commits形式のコミットメッセージを生成する。
    ユーザーが「コミット」「commit」「変更を保存」「git commit」等と言ったときに使用する。引数なしで最後の変更内容を自動検出する。
---

# コミットメッセージの生成スキル

## ルール

Conventional Commits形式でコミットメッセージを生成する。

**Defaultブランチ（main等）での保護:** Defaultブランチ検出時、警告を表示し、新規ブランチ作成またはそのまま続行を選択させる。

フォーマット、type、scope の定義については [conventions.md](conventions.md) を参照してください。

## 手順

1. `git rev-parse --abbrev-ref HEAD` を実行して現在のブランチを確認
2. Defaultブランチ（main, master等）の場合、警告を表示
3. `git diff --staged` を実行して変更内容を確認する
4. 変更の性質を分析し、typeを選択する
5. scope を決定（詳細は conventions.md 参照）：複数ファイルか単一ファイルか、影響範囲が明確か判定して選択
6. 変更内容に破壊的な変更が含まれ場合、 `<type>(<scope>)` の後に `!` を付け加える（scope がない場合は `<type>!`）
7. 変更内容を50文字以内のsubjectにまとめる
8. 必要に応じてbodyに詳細を追加する
9. コミットを実行する

## 参考

動作例については、`examples/sample.md` を参照してください。
