# macOS CLI 実装規約

## 適用条件

macOS 向け CLI 実行ファイルで、以下のいずれかを含むソースに適用する。

- signal handler の登録（`signal(2)` / `sigaction(2)` / `@convention(c)` ハンドラー）
- ターミナルの raw mode 操作（`termios` / `tcsetattr` / `tcgetattr`）

**他ルールとの優先順位:** 本規約は `swift-concurrency.md` の `nonisolated(unsafe)`
使用禁止条項に優先する。signal handler が参照するグローバルには `nonisolated(unsafe)` が
必要であり、その使用可否は本ファイルの規定に従う。

---

## Signal Handler

Swift CLI で SIGINT / SIGTERM などを捕捉する場合の実装規約。

### 制約: async-signal-safe 関数のみ使用可

signal handler はカーネルが任意のタイミングで割り込んで呼び出す。
ハンドラー内で使える関数は **POSIX async-signal-safe** に限定される。

**禁止（signal handler 内で呼んではいけないもの）:**
- `malloc` / `free`（Swift ARC の retain/release を含む）
- Swift ランタイム全般（`print`, `String` 生成, クロージャコンテキスト参照）
- Objective-C メッセージ送信（`NSLog` 含む）
- `NSLock`, `DispatchQueue`, `DispatchSemaphore`

**許可（async-signal-safe の例）:**
- `tcsetattr` / `tcgetattr`
- `signal(2)`
- `kill(2)`
- `write(2)`（ファイルディスクリプタへの低レベル書き込み）
- `_exit(2)`

```swift
// 良い例: async-signal-safe 関数のみ使用
private let _sigintHandler: @convention(c) (Int32) -> Void = { _ in
    if let ptr = _originalTermiosStorage {
        _ = tcsetattr(STDIN_FILENO, TCSAFLUSH, ptr)  // OK: async-signal-safe
    }
    _ = Darwin.signal(SIGINT, SIG_DFL)               // OK: async-signal-safe
    kill(getpid(), SIGINT)                            // OK: async-signal-safe
}

// 悪い例: ハンドラー内で Swift ランタイムを呼ぶ
private let _badHandler: @convention(c) (Int32) -> Void = { _ in
    print("SIGINT received")  // ❌ Swift ランタイム・malloc を使う
}
```

### `@convention(c)` クロージャの制限

C API（`sigaction`）にハンドラーを渡すには `@convention(c)` 属性が必要。

- `@convention(c)` クロージャは**コンテキストのキャプチャ不可**
- 外部の値を参照する場合はグローバル変数経由にする

```swift
// 良い例: グローバル変数でキャプチャを回避
nonisolated(unsafe) private var _originalTermiosStorage: UnsafeMutablePointer<termios>? = nil

private let _sigintHandler: @convention(c) (Int32) -> Void = { _ in
    if let ptr = _originalTermiosStorage { ... }  // グローバル経由でアクセス
}

// 悪い例: クロージャでローカル変数をキャプチャ（コンパイルエラー）
func setup(original: termios) {
    let _badHandler: @convention(c) (Int32) -> Void = { _ in
        // original を参照しようとするとコンパイルエラー ❌
    }
}
```

### `nonisolated(unsafe)` グローバルへのアクセス

Swift 6 strict concurrency 環境では、`@convention(c)` ハンドラーがアクセスする
グローバル変数に `nonisolated(unsafe)` が必要。

- `nonisolated(unsafe)` は「Swift ランタイムの並行安全保証なしでアクセスする」という宣言
- 使用箇所を**最小限**にし、理由を必ずコメントで明記する
- 書き込みは必ず単一スレッドまたは排他制御下で行う

```swift
// Heap-allocated storage for original termios.
// Accessed from a @convention(c) signal handler, so must be global and nonisolated(unsafe).
nonisolated(unsafe) private var _originalTermiosStorage: UnsafeMutablePointer<termios>? = nil
```

### 状態のヒープ割り当て

`@convention(c)` ハンドラーは呼び出し元のスタックフレームにアクセスできない。
ハンドラーが参照する状態は**ヒープに割り当てて**グローバルポインタ経由でアクセスする。

```swift
// 良い例: ヒープに割り当ててグローバルポインタ経由でアクセス
let ptr = UnsafeMutablePointer<termios>.allocate(capacity: 1)
ptr.initialize(to: original)
_originalTermiosStorage?.deallocate()  // 古いポインタを解放してから
_originalTermiosStorage = ptr

// 解除時は必ず deallocate する
if let ptr = _originalTermiosStorage {
    _ = tcsetattr(STDIN_FILENO, TCSAFLUSH, ptr)
    _originalTermiosStorage = nil
    ptr.deallocate()  // ← 忘れるとメモリリーク
}
```

### `sigaction(2)` vs `signal(2)` の選択

`signal(2)` はシグナル処理後にハンドラーがデフォルト（`SIG_DFL`）に戻る実装系があるため、
**`sigaction(2)` を使用する**。

Darwin では `sigaction` という名前の struct と function が共存するため、
`typealias` で関数参照を明示する。

```swift
// Darwin 固有: 同名の struct と function を区別する
private typealias SigactionFn = (Int32, UnsafePointer<Darwin.sigaction>?, UnsafeMutablePointer<Darwin.sigaction>?) -> Int32

// 登録
var action = Darwin.sigaction()
action.__sigaction_u = __sigaction_u(__sa_handler: _sigintHandler)
sigemptyset(&action.sa_mask)
action.sa_flags = 0
withUnsafePointer(to: action) { ptr in
    let fn: SigactionFn = sigaction
    _ = fn(SIGINT, ptr, nil)
}
```

## Terminal / Raw Mode

ターミナルを raw mode にする際の注意点（signal handler と組み合わせる場合が多い）。

- raw mode 有効化前に元の `termios` を保存し、終了時・シグナル受信時に必ず復元する
- `isatty(STDIN_FILENO)` で TTY 判定してから raw mode に入る（CI/パイプ環境で誤作動防止）
- `tcsetattr` は `TCSAFLUSH` を使い、既存の入力バッファをフラッシュしてから設定する

```swift
// TTY 判定
guard isatty(STDIN_FILENO) != 0 else { return }  // 非 TTY ならスキップ

// 元の設定を保存
var original = termios()
guard tcgetattr(STDIN_FILENO, &original) == 0 else { ... }

// raw mode に設定
var raw = original
raw.c_lflag &= ~tcflag_t(ICANON | ECHO | ISIG | IEXTEN)
raw.c_iflag &= ~tcflag_t(IXON | ICRNL | BRKINT | INPCK | ISTRIP)
raw.c_oflag &= ~tcflag_t(OPOST)
raw.c_cflag |= tcflag_t(CS8)
guard tcsetattr(STDIN_FILENO, TCSAFLUSH, &raw) == 0 else { ... }
```

---

## レビューチェックリスト

**signal handler:**

- [ ] ハンドラー内で Swift ランタイム（ARC、`print`、`String` 生成）を呼んでいないか
- [ ] `@convention(c)` クロージャがコンテキストをキャプチャしていないか
- [ ] `nonisolated(unsafe)` 変数の使用箇所に理由コメントがあるか
- [ ] ヒープポインタが使用後に `deallocate()` されているか
- [ ] `sigaction` でハンドラーをリストア（解除）しているか
- [ ] `signal(2)` ではなく `sigaction(2)` を使っているか

**raw mode:**

- [ ] raw mode 有効化前に元の `termios` を保存しているか
- [ ] 終了時とシグナル受信時の両方で `termios` を復元しているか
- [ ] `isatty(STDIN_FILENO)` で TTY 判定してから raw mode に入っているか
- [ ] `tcsetattr` に `TCSAFLUSH` を使っているか
