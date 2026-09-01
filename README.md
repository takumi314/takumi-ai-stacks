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

| ルール名 | 説明 | 適用対象 |
| --- | --- | --- |
| [macos-cli](rules/macos-cli.md) | signal handler の async-signal-safe 制約、`@convention(c)` の扱い、termios の復元を規定する | macOS CLI で signal handler・raw mode を扱うソース |
| [security](rules/security.md) | シークレット管理・通信・入力検証・ログ出力・依存関係の基本要件を規定する | 言語・プラットフォームを問わず全プロジェクト |
| [sourcekit-lsp-troubleshooting](rules/sourcekit-lsp-troubleshooting.md) | エディタ診断とビルド結果が食い違う場合に、何を実行してから判断するかを規定する | SPM + SourceKit-LSP の診断表示 |
| [swift](rules/swift.md) | 命名・インデント・アクセス制御・型安全性など Swift の基本規約を規定する | `.swift` ファイル全般 |
| [swift-clean-code](rules/swift-clean-code.md) | 命名の質・関数の責務や抽象度・データ構造設計など、関数/型内部に閉じる可読性の観点を規定する | `.swift` ファイル全般（命名・関数分割・データ設計のレビュー観点） |
| [swift-concurrency](rules/swift-concurrency.md) | `Sendable` 適合、`@MainActor` 型の deinit、`nonisolated` の境界、GCD からの移行を規定する | Swift 6 言語モードの並行コード |
| [swift-error](rules/swift-error.md) | `throws` / `Result` / `Optional` の使い分けと禁止パターンを規定する | エラーが発生しうる処理全般 |
| [swift-solid](rules/swift-solid.md) | 継承よりコンポジション、条件分岐よりポリモーフィズム、SOLID 各原則など型同士の関係レベルの設計判断を規定する | `class` / `struct` / `protocol` の新規設計・既存型階層の変更 |
| [swift-syntax](rules/swift-syntax.md) | `Sendable` 境界、offset 整合、置換テキストの構文非破壊検査を規定する | SwiftSyntax / SwiftParser を使うソース |
| [swift-test](rules/swift-test.md) | Swift Testing の使用、テスト命名、AAA パターン、モック設計を規定する | `Tests/` 配下のテストコード |
| [swiftui-view](rules/swiftui-view.md) | View の分割、Asset Catalog による Color 指定、Text Style、State 管理を規定する | SwiftUI View の実装 |
| [viewmodel](rules/viewmodel.md) | `@MainActor` + `ObservableObject`、`enum State` による状態管理、DI を規定する | MVVM 構成の ViewModel |

ルールを追加したら、この表に1行追記してください。

## 新しい skill / rule を追加する際の格納ルール

### skill の追加

- `skills/<skill-name>/` ディレクトリを作成し、最低限 `SKILL.md` を配置する
- `SKILL.md` の frontmatter（`name`, `description`）を正しく記述し、`description` にはどんな
  ユーザー発話で起動すべきかを具体的に書く
- 追加したら本 README の [skill 一覧](#skill-一覧) に1行追記する
- 詳細な構成規約（`conventions.md` や `examples/` の使い分けなど）は [CLAUDE.md](CLAUDE.md) を参照

### rules の追加

- `rules/<rule-name>.md` の形式で、1ファイル1トピックの構成で追加する
- `~/.claude/rules/` に置いたルールは**全ファイルが毎セッション自動でコンテキストに載る**。
  無関係な場面で誤適用されないよう、h1 直下に `## 適用条件` を必ず置き、
  適用対象と、他ルールと競合した場合にどちらが優先されるかを明記する
- 末尾に `## レビューチェックリスト` を `- [ ]` 形式で置く。
  CLAUDE.md が求める「PR 前の自己照合」は、チェックリストが無いと実行できない
- 「心がける」「必要に応じて」「望ましい」など違反を判定できない表現は使わない。
  判定可能な条件（何をしたら違反か）に置き換える
- 禁止事項には必ず ✅ の代替手段を併記する
- Swift のコード例は掲載前に検証する。✅ 例が通り、❌ 例がエラーになることを確認する：

  ```sh
  printf '%s' '<コード例>' | swiftc -typecheck -swift-version 6 -
  ```

- 追加したら本 README の [rules 一覧](#rules-一覧) に追記する

## 今後の方針

今後 `skills/` や `rules/` に活用できるものを追加し、内容を充実させていく予定です。
