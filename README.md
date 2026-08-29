# takumi-ai-stacks

Claude Code 用の skill 定義・ルールをストックし、バージョン管理するためのリポジトリです。

## 利用方法

`skills/*` は `~/.claude/skills/` に、`rules/*` は `~/.claude/rules/` に配置して使う想定です。
現状はファイルコピーによる手動同期の運用です（symlink ではありません）。

```sh
# 必要な skill だけをコピーする例
cp -r skills/auto-commit-msg ~/.claude/skills/

# 必要な rule だけをコピーする例
cp rules/<rule-name>.md ~/.claude/rules/
```

新しい環境でこのリポジトリを使い始める場合は、リポジトリを clone した上で、上記のように必要な
skill / rule だけを `~/.claude/` 配下にコピーしてください。全体をまとめてコピーしても構いません。

## skill 一覧

| skill 名 | 説明 | 利用場面 |
| --- | --- | --- |
| [auto-commit-msg](skills/auto-commit-msg/SKILL.md) | Conventional Commits 形式のコミットメッセージを自動生成する | 「コミットして」「commit」「変更を保存」等と言われたとき |

skill を追加したら、この表に1行追記してください。

## rules 一覧

現時点で `rules/` は空です。今後ルールを追加した際は、skill 一覧と同様にこのセクションへ
一覧テーブルを追加していきます。

## 新しい skill / rule を追加する際の格納ルール

### skill の追加

- `skills/<skill-name>/` ディレクトリを作成し、最低限 `SKILL.md` を配置する
- `SKILL.md` の frontmatter（`name`, `description`）を正しく記述し、`description` にはどんな
  ユーザー発話で起動すべきかを具体的に書く
- 追加したら本 README の [skill 一覧](#skill-一覧) に1行追記する
- 詳細な構成規約（`conventions.md` や `examples/` の使い分けなど）は [CLAUDE.md](CLAUDE.md) を参照

### rules の追加

- `rules/<rule-name>.md` の形式で、1ファイル1トピックの構成で追加する
- 追加したら本 README の [rules 一覧](#rules-一覧) に追記する

## 今後の方針

今後 `skills/` や `rules/` に活用できるものを追加し、内容を充実させていく予定です。
