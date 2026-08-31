# Swift エラー表現規約

Swift におけるエラーの表現方法を統一する規約。
「どの状況でどのメカニズムを選ぶか」を迷わず決定できるようにすることが目的。

---

## 適用条件

エラーが発生しうる処理を実装・変更するすべての Swift ソースに適用する。

**他ルールとの優先順位:** 本ファイルはエラー処理に特化した補足規約であり、
命名・インデント・アクセス制御などの一般規約は `swift.md` に従う。
非同期処理と組み合わせる場合は `swift-concurrency.md` を併せて参照する。

---

## エラー表現の選択基準

| 状況 | 使うもの |
|------|----------|
| 処理が失敗しうる同期関数 | `throws` |
| 処理が失敗しうる非同期関数 | `async throws` |
| エラーを値として渡す・保持する必要がある | `Result<T, Error>` |
| 値の「不在」を表す（エラーではない） | `T?` / `Optional` |
| コールバック API を async/await に橋渡しする | `withCheckedThrowingContinuation` |
| 絶対に失敗しないが型が Optional な初期化 | `!` は禁止 → `guard let` + fatalError でコメント必須 |

---

## `throws` を使う

同期・非同期を問わず、**呼び出し元がエラーを認識して対処できる**場合は `throws` を使う。

```swift
// 良い例
func parse(at url: URL) throws -> ContentsJSON {
    let data = try Data(contentsOf: url)
    return try JSONDecoder().decode(ContentsJSON.self, from: data)
}

// 良い例（非同期）
func resize(file: URL, toWidth w: Int, height h: Int) async throws {
    let result = try await runProcess(...)
    guard result.exitCode == 0 else {
        throw SipsError.resizeFailed(file, stderr: result.stderr)
    }
}
```

---

## `Result<T, Error>` を使う

以下の場合に限り `Result` を使う。`throws` で代替できる場合は `throws` を優先する。

- クロージャ引数でエラーを返す（`@escaping` completion handler の廃止移行期）
- エラーを配列に蓄積して後でまとめて返す

```swift
// 良い例: エラーをまとめて収集する
func validateAll(entries: [ImageEntry]) -> [Result<IconResult, IconIssue>] { ... }

// 悪い例: throws で書けるのに Result を使っている
func parse(url: URL) -> Result<ContentsJSON, Error> { ... }  // throws にすること
```

---

## `Optional` を使う

エラーではなく「値がない状態が正常」な場合にのみ使う。

```swift
// 良い例: size/scale の欠落は仕様上あり得る → Optional
func pixelSize(from entry: ImageEntry) -> PixelSize? { ... }

// 悪い例: エラーが起きたことを nil で隠蔽している
func resize(file: URL) -> URL? { ... }  // 失敗理由が呼び出し元に伝わらない
```

---

## カスタムエラー型の定義

`enum` + `Error` 準拠を基本とする。ユーザー向けメッセージが必要な場合は `LocalizedError` も採用する。

```swift
// 基本形
enum IconIssue: Error, Equatable, Sendable {
    case fileNotFound
    case notPNG(actual: String)
    case wrongWidth(expected: Int, actual: Int)
    case hasAlphaChannel
}

// ユーザー向けメッセージが必要な場合
enum SipsError: LocalizedError, Sendable {
    case resizeFailed(URL, stderr: String)

    var errorDescription: String? {
        switch self {
        case .resizeFailed(let url, let stderr):
            return "リサイズ失敗: \(url.lastPathComponent) — \(stderr)"
        }
    }
}
```

---

## 禁止パターン

### 戻り値でエラーを表現する（最重要禁止）

```swift
// ❌ 禁止: 成功時は結果文字列、失敗時はエラーメッセージ文字列
func fixAlpha(file: URL) async -> String { ... }

// ✅ 正しい: 成功時の説明文字列と失敗を型で分離
func fixAlpha(file: URL) async throws -> String { ... }
// または
func fixAlpha(file: URL) async -> Result<String, FixError> { ... }
```

呼び出し元が「エラーかどうか」を文字列のプレフィックス（`"エラー:"` など）で判断しなければならない設計は禁止。

### `try!`

```swift
// ❌ 禁止
let data = try! Data(contentsOf: url)

// ✅ テストコードでも guard + XCTFail / Issue.record を使う
guard let data = try? Data(contentsOf: url) else {
    Issue.record("ファイル読み込み失敗: \(url)")
    return
}
```

### `try?` の乱用

```swift
// ❌ エラーを握りつぶす try?（デバッグ不能になる）
let contents = try? parser.parse(at: url)

// ✅ エラーを伝播させる
let contents = try parser.parse(at: url)

// ✅ 失敗を想定内として扱う場合は意図をコメントで明記
// バックアップが既に存在しない場合は無視してよい
try? FileManager.default.removeItem(at: backup)
```

### デコード時の `try?` によるフィールド握りつぶし

```swift
// ❌ images フィールドが壊れていてもエラーにならない
self.images = (try? container.decode([ImageEntry].self, forKey: .images)) ?? []

// ✅ フィールド欠落と型不正を区別する
self.images = container.contains(.images)
    ? try container.decode([ImageEntry].self, forKey: .images)
    : []
```

---

## `withCheckedThrowingContinuation` でのエラー処理

すべてのコードパスで `resume` が **必ず1回** 呼ばれることを保証する。

```swift
func runProcess(...) async throws -> ProcessResult {
    try await withCheckedThrowingContinuation { continuation in
        let process = Process()
        process.terminationHandler = { proc in
            // ここで必ず resume される
            continuation.resume(returning: ProcessResult(...))
        }
        do {
            try process.run()
        } catch {
            // run() が失敗した場合も必ず resume される
            continuation.resume(throwing: error)
        }
        // 注意: terminationHandler が呼ばれない分岐を作らないこと
    }
}
```

---

## エラー伝播 vs その場でハンドリング

| 判断基準 | 対処 |
|----------|------|
| 呼び出し元が回復処理を持てる | `throws` で伝播させる |
| ここで回復するしかない（ロールバック等） | `do-catch` でハンドリング |
| 失敗してもシステム全体に影響しない後処理 | `try?`（意図コメント必須） |
| 絶対に起きてはならない前提条件の破壊 | `fatalError("理由")` または `preconditionFailure` |

---

## レビューチェックリスト

- [ ] 失敗しうる処理が `throws` / `async throws` で表現されているか
- [ ] `throws` で書けるのに `Result` を使っていないか
- [ ] Optional が「値の不在が正常」な場合に限って使われているか（失敗の隠蔽になっていないか）
- [ ] 戻り値の文字列やプレフィックスでエラーを表現していないか
- [ ] `try!` が使われていないか（テストコードを含む）
- [ ] `try?` に、失敗を無視してよい理由のコメントがあるか
- [ ] デコード処理で `try?` によりフィールドの型不正を握りつぶしていないか
- [ ] カスタムエラー型が `enum` + `Error` 準拠で定義されているか
- [ ] ユーザー向けメッセージが必要な型が `LocalizedError` に準拠しているか
- [ ] `withCheckedThrowingContinuation` の全コードパスで `resume` が必ず 1 回呼ばれるか
