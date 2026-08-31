# Swift コーディング規約

Swift コードの基本規約。命名・インデント・アクセス制御など、
すべての Swift ソースに共通する事項を定める。

---

## 適用条件

`.swift` ファイルを新規作成・編集するすべての場面に適用する。

**他ルールとの優先順位:** 本規約は最も適用範囲の広い一般ルールであり、
より適用範囲の狭いルールが競合する場合はそちらを優先する。

| 領域 | 優先されるルール |
|------|----------|
| エラー表現（`throws` / `Result` / `Optional`） | `swift-error.md` |
| 並行性（`@MainActor` / `actor` / `Task` / `Sendable`） | `swift-concurrency.md` |
| SwiftUI View | `swiftui-view.md` |
| ViewModel | `viewmodel.md` |
| テストコード | `swift-test.md` |
| macOS CLI（signal handler / raw mode） | `macos-cli.md` |
| swift-syntax / SwiftParser | `swift-syntax.md` |

---

## インデント

- **iOS 実装（Swift、Objective-C）**: 4 スペース（Xcode 標準）
- **その他のプロジェクト**: 2 スペース

---

## 命名規則

- **型名（クラス、構造体、列挙型）**: `PascalCase`（例：`ContentView`、`UserViewModel`）
- **変数・定数・関数**: `camelCase`（例：`backgroundColor`、`fetchUserData()`）
- **定数**（トップレベル）: `UPPER_SNAKE_CASE`（例：`MAX_RETRY_COUNT`）
- **プライベート変数**: 先頭に `_` を付けない（`private let userData` と書く）

---

## 非同期処理

iOS 開発では **Swift Concurrency（async/await）** を使用する。

- 新規コードで `completion handler` を定義しない。
  既存の completion handler API を呼ぶ場合は `withCheckedThrowingContinuation` で
  async 関数に橋渡しする（`swift-error.md`）
- `Combine` は、SwiftUI の `@Published` / `ObservableObject` および
  Combine を前提とする既存 API との接続に限って使用する。
  新規の非同期処理を Combine で書かない
- `@MainActor` / `actor` / `Sendable` を含む詳細は `swift-concurrency.md` に従う

---

## アクセス制御

- デフォルトは `internal`（アクセス修飾子を書かない）
- 型外部から参照されないメンバーには `private` を付ける
- モジュール外へ公開する API にのみ `public` を付ける

---

## 型安全性

- Force unwrap（`!`）を使用しない。`guard let` / `if let` でアンラップする
- 強制キャスト（`as!`）を使用しない。`as?` + `guard let` を使う
- Optional を返すのは「値の不在が正常な状態」の場合に限る。
  失敗を表す場合は `throws` を使う（`swift-error.md`）

---

## エラーハンドリング

- 失敗しうる処理は `throws` で明示する
- `try!` を使用しない。`try?` はエラーを無視してよい理由をコメントに書ける場合のみ使う
- 詳細な使い分けは `swift-error.md` に従う

---

## コメント

- 「なぜそう書いたか」が自明でない箇所にコメントを書く（コードが何をしているかは書かない）
- 日本語コメント可

---

## その他

- 1 ファイル = 1 型定義（ネストした型・`extension` は同一ファイルに置いてよい）
- 同じロジックが 2 箇所以上に現れたら関数・メソッドに抽出する
- 外部依存（ネットワーク・ファイル・時刻）は Protocol 経由で注入し、
  テストから差し替え可能にする（`swift-test.md`）

---

## レビューチェックリスト

- [ ] インデント幅がプロジェクトの規定（iOS は 4、その他は 2）に揃っているか
- [ ] 型名・変数名・トップレベル定数の命名規則に従っているか
- [ ] プライベート変数の先頭に `_` が付いていないか
- [ ] 新規コードで completion handler を定義していないか
- [ ] Combine の使用が `@Published` / 既存 API 接続の範囲に収まっているか
- [ ] 型外部から参照されないメンバーに `private` が付いているか
- [ ] Force unwrap（`!`）・強制キャスト（`as!`）が使われていないか
- [ ] `try!` が使われていないか。`try?` に理由コメントがあるか
- [ ] 失敗を Optional で表現していないか（`throws` を使うべきでないか）
- [ ] 1 ファイルに複数の型定義が混在していないか
- [ ] 同一ロジックの重複が 2 箇所以上残っていないか
- [ ] 外部依存が Protocol 経由で注入され、テストから差し替え可能か
