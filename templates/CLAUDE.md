# CLAUDE.md

> このファイルは [takumi-ai-stacks](https://github.com/) の `templates/CLAUDE.md` をコピーして
> 作成するテンプレートです。実プロジェクトのルートに `CLAUDE.md` として配置し、`<...>` の
> プレースホルダー箇所を実際の値に書き換えてください。

このファイルは、Claude Code がこのリポジトリで作業する際に最初に読み込む前提ルールです。

## iOS Project Guide

- **アプリ名 / 目的**: `<アプリ名>` — `<このアプリが解決する課題を1〜2文で>`
- **アーキテクチャ**: MVVM（Model-View-ViewModel）パターン、SPM によるモジュール分割
- **UI**: SwiftUI 優先、Swift Concurrency（async/await）を使用
  （詳細な実装規約は `~/.claude/rules/swift.md`、`swiftui-view.md`、`viewmodel.md` を参照）
- **モジュール構成**:
  - `Sources/<ModuleName1>` — `<責務を1行で>`
  - `Sources/<ModuleName2>` — `<責務を1行で>`
  - （実際のモジュール一覧に置き換える）

## 開発環境

- **Xcode**: `<Xcode バージョン、例: 16.x>`
- **Swift**: `<Swift バージョン、例: 6.0>`
- **依存管理**: Swift Package Manager（SwiftPM）。CocoaPods 等は使用しない
- **実装規約**: `~/.claude/rules/` 配下の各ファイルに従う。
  **実装後に PR を作る前には必ず該当ファイルを読んで自己照合すること。**
  - `swift.md` / `swift-clean-code.md` / `swift-solid.md` — コーディング規約・設計
  - `swift-error.md` — エラー表現（throws / Result / Optional）
  - `swiftui-view.md` / `viewmodel.md` — SwiftUI View / ViewModel 実装規約
  - `swift-concurrency.md` — Swift Concurrency（Sendable・@MainActor）
  - `swift-test.md` — テスト実装規約
  - `security.md` — セキュリティ要件

## CRITICAL RULES (Strict Enforcement)

コーディング規約そのものは上記の `~/.claude/rules/` に従う。ここではプロセス面で
**例外なく守るべき事項**のみを定める。

1. **実装前後に該当する `~/.claude/rules/*.md` を読み、自己照合してから完了報告する。**
   自己照合を省略した「実装しました」報告は禁止
2. **ビルドが通らない状態でコミット・PR作成をしない。**（詳細は下記 Workflow Hook）
3. **破壊的な git 操作**（`push --force`、`reset --hard`、`checkout .`、`clean -f` 等）は、
   ユーザーの明示的な承認なしに実行しない
4. **開発環境・ビルド設定を無断で変更しない。** `Info.plist`、`.xcodeproj` の Signing/Build
   Settings、`Package.swift` の `platforms` 等を変更する場合は、変更内容と理由を事前に説明し、
   承認を得てから行う
5. **シークレット情報**（API キー、トークン、証明書等）をコードやコミットに含めない。
   `.gitignore` で管理されたファイル（`.env` 等）から読み込む
6. **エディタの診断表示ではなく `swift build` / `swift test` の実行結果を唯一の真実とする。**
   診断だけを根拠にコードを変更しない
   （詳細は `~/.claude/rules/sourcekit-lsp-troubleshooting.md`）

## Build and Test Commands

> 実際のプロジェクト構成（SPM のみ / Xcode プロジェクトあり）に応じて書き換えること。

```sh
# ビルド
swift build

# 全テスト実行
swift test

# 単一テストの実行（Swift Testing）
swift test --filter <TestSuiteName>/<testMethodName>

# Xcode プロジェクトがある場合のビルド例
xcodebuild -scheme "<SchemeName>" -destination "platform=iOS Simulator,name=<SimulatorName>" build

# Xcode プロジェクトがある場合のテスト例
xcodebuild -scheme "<SchemeName>" -destination "platform=iOS Simulator,name=<SimulatorName>" test
```

## Workflow Hook（ビルド破壊防止の手順）

「勝手にコードを書き換えて、ビルドが通らなくなる」事態を防ぐため、コード変更時は必ず
以下の手順を踏む。

1. `.swift` ファイルを1つでも編集したら、**完了報告の前に必ず** 上記の `swift build`
   （テスト対象の変更なら `swift test`）を実行する
2. ビルド・テストが失敗した場合、その場でエラーを解消してから次の作業に進む。
   エラーを未解決のまま「実装完了」と報告しない
3. 複数ファイルにまたがる変更をした場合、個々のファイル単位ではなく、最後に
   **プロジェクト全体のビルド**を通してから完了とする
4. ビルドが通らないままユーザーに引き渡す必要がある場合（環境依存の失敗など）は、
   「ビルドが通っていない」ことと、その原因・再現手順を明示的に報告する。
   黙って完了扱いにしない
