# Swift Concurrency 実装規約

Swift 6 の厳格な並行性チェック（Strict Concurrency Checking）に準拠し、
データ競合のない非同期コードを書くための規約。

---

## 適用条件

以下を **すべて** 満たす場合に適用する。

- Swift 6 言語モードが有効（SPM: `swiftLanguageModes: [.v6]` / Xcode: Swift Language Version = 6）
- `@MainActor` / `actor` / `Task` / `Sendable` / `AsyncStream` のいずれかを含むソース

Swift 5 言語モードでは同じコードでも診断が警告に留まり挙動が異なるため、本規約は適用しない。

**検証環境:** Swift 6.3。本ファイルのコード例はすべて
`swiftc -typecheck -swift-version 6` で挙動を確認済み。

**他ルールとの優先順位:** 本規約が他ルールと競合する場合、適用範囲の狭い方を優先する。

- signal handler 内の `nonisolated(unsafe)` は `macos-cli.md` を優先する
- 一般的な非同期方針は `swift.md`、ViewModel 固有の規約は `viewmodel.md` を参照する

---

## 基本原則

- アクター境界を越える値は `Sendable` に適合させる（適合のさせ方は次節）
- すでに非同期コンテキストにいる場合、`Task { ... }` を新たに作らない
- `Task` がインスタンスの生存期間を超えて動き続ける場合は `[weak self]` を使う

`Task { ... }` は完了時に `self` を解放するため、それ自体は循環参照ではない。
`[weak self]` が必要なのは **解放が Task 完了まで遅延すると困る場合** に限る。
具体的には、`for await` で無限ストリームを consume する Task、
`while true` を含む Task、インスタンスより長生きするプロパティに保持した Task。

```swift
// ✅ 長寿命 Task: 解放遅延を避けるため [weak self]
func observe(_ events: AsyncStream<Event>) -> Task<Void, Never> {
    Task { [weak self] in
        for await event in events {
            guard let self else { return }
            await self.handle(event)
        }
    }
}

// ✅ 短命 Task: 完了時に解放されるため [weak self] は不要
func reload() {
    Task { await self.fetch() }
}
```

---

## `Sendable` への適合

上から順に判定し、最初に当てはまるものを採用する。

| 型の性質 | 適合のさせ方 |
|------|----------|
| 全格納プロパティが `Sendable` な `struct` / `enum` | `: Sendable`（多くは自動導出される） |
| `let` のみを持つ `class` | `final class X: Sendable` |
| 可変状態を自前のロックで守る `class` | `: @unchecked Sendable` + ロック対象をコメント必須 |
| 可変状態を非同期に共有する | `actor` にする |
| 上記のいずれにも当てはまらない | `Sendable` にせず、アクター境界を越えさせない |

`@unchecked Sendable` は「どのロックで何を守っているか」をコメントに書けない場合は使用しない。
書けないということは不変条件が定まっていないということであり、その状態で付けると
コンパイラの検査を無効化するだけになる。

---

## `@MainActor` 型の deinit

`deinit` は `nonisolated` である。触れてよい範囲は以下のとおり厳密に決まっている。

| `deinit` からの操作 | 可否 |
|------|----------|
| 格納プロパティへの直接アクセス | ✅ 可能 |
| 分離メソッドの呼び出し | ❌ `call to main actor-isolated instance method ... in a synchronous nonisolated context` |
| 計算プロパティの参照 | ❌ `main actor-isolated property ... can not be referenced from a nonisolated context` |
| `Task { }` を作って `self` に触れる | ❌ `capture of 'self' in a closure that outlives deinit` |

```swift
import Foundation

@MainActor final class ViewModel {
    var timer: Timer?
    var label: String { "vm" }

    func stop() { timer?.invalidate() }

    deinit {
        // ✅ 格納プロパティへの直接アクセスは可能
        timer?.invalidate()

        // ❌ 分離メソッドは呼べない
        // stop()

        // ❌ 計算プロパティは参照できない
        // _ = label

        // ❌ Task で囲んでも解決しない。self をキャプチャできずコンパイルエラーになる。
        //    仮に回避できても解放が Task 完了まで遅延し、deinit の意味を失う。
        // Task { @MainActor in self.timer?.invalidate() }
    }
}
```

格納プロパティへの直接アクセスで足りない場合は、以下の順で選ぶ。

1. **`isolated deinit` を使う**（SE-0371 / Swift 6.1 以降）。分離メソッドも計算プロパティも扱える。

   ```swift
   @MainActor final class ViewModel2 {
       var timer: Timer?
       func stop() { timer?.invalidate() }

       isolated deinit { stop() }   // ✅ メインアクター上で実行される
   }
   ```

2. **`deinit` に頼らない。** Swift 6.1 未満のツールチェーンを対象にする場合、
   明示的な `cancel()` / `cleanup()` を用意し、呼び出し側の責務として実行する。
   破棄タイミングをライフサイクル（`onDisappear`、`task` のキャンセル等）に紐付ける。

---

## システムコールバックと `@MainActor` クロージャ推論

Swift 6 では、`@MainActor` メソッド内で定義したクロージャは、`self` をキャプチャしていなくても
コンパイラが `@MainActor` を推論する。`AVAudioEngine` の `installTap`、`AVCaptureSession` の
出力コールバックなど、**リアルタイム・バックグラウンドスレッドから呼ばれるシステムコールバック**
でこのパターンが発生すると `dispatch_assert_queue_fail` でクラッシュする。

**コンパイルは通る。失敗はランタイムにしか現れない**ため、実装時に意識して避ける必要がある。

```swift
// ❌ @MainActor メソッド内でクロージャを定義
//    → Swift 6 が @MainActor を推論 → リアルタイムスレッドから呼ばれてクラッシュ
@MainActor func startTransmitting() async throws {
    inputNode.installTap(onBus: 0, bufferSize: 1024, format: nil) { buffer, _ in
        continuation?.yield(.chunkReady(data))  // dispatch_assert_queue_fail
    }
}

// ✅ @MainActor なしのプライベートヘルパーに抽出し、値をパラメーターで渡す
@MainActor func startTransmitting() async throws {
    installTapHelper(on: inputNode, continuation: lock.withLock { continuation })
}

// @MainActor アノテーションなし → クロージャも非分離
private func installTapHelper(
    on inputNode: AVAudioInputNode,
    continuation: AsyncStream<Event>.Continuation?
) {
    inputNode.installTap(onBus: 0, bufferSize: 1024, format: nil) { buffer, _ in
        continuation?.yield(.chunkReady(data))  // 非分離: リアルタイムスレッドから安全
    }
}
```

**適用範囲:**

- `AVAudioEngine.installTap` / `removeTap`
- `AVCaptureOutput.setSampleBufferDelegate`
- `DispatchSource` / `DispatchQueue` のハンドラ
- 外部 C コールバック（AudioUnit render コールバック等）

---

## `nonisolated` との境界

- アクターの状態に依存しない計算プロパティ・メソッドには `nonisolated` を付ける。
  **判定基準:** 本体が `self` の可変格納プロパティを一切読まないこと。
  `let` のみを読む場合は `nonisolated` にできる。

  ```swift
  // ✅ let のみを読むので nonisolated にできる（不要なアクターホップを避けられる）
  actor Downloader {
      let id: String
      var progress: Double = 0

      nonisolated var label: String { "Downloader(\(id))" }
  }
  ```

- **`nonisolated(unsafe)` は使用しない。** 例外は以下の 2 つに限る。

  1. signal handler から参照するグローバル（`macos-cli.md` の規定に従う）
  2. 外部 C / Objective-C ライブラリが要求するグローバル

  例外を使う場合、**どの不変条件を人手で保証しているか**をコメントに必ず書く。
  書けない場合は使用せず、`actor` またはロック付き `@unchecked Sendable` に置き換える。

---

## `DispatchQueue.main` と Swift Concurrency の混在

`DispatchQueue.main.async` より `@MainActor` を優先する。

| 状況 | 使うもの |
|------|----------|
| 新規実装 | メソッドに `@MainActor` を付与する |
| 非同期関数から一時的にメインへ戻る | `await MainActor.run { ... }` |
| 同期関数からメインアクターを呼ぶ | `Task { @MainActor in ... }` |
| レガシー API の完了コールバック内で UI 更新 | `Task { @MainActor in ... }` |
| メインスレッド上と静的に分かっている同期文脈 | `MainActor.assumeIsolated { ... }` |

```swift
// ✅ 非同期関数から一時的にメインへ戻る
func refresh(vm: ViewModel) async {
    let items = await loadItems()
    await MainActor.run { vm.items = items }
}

// ❌ Swift Concurrency の文脈で GCD に戻す
func refreshBad(vm: ViewModel) async {
    let items = await loadItems()
    DispatchQueue.main.async { vm.items = items }
}
```

`MainActor.assumeIsolated` はメインスレッド上であることが保証できる場合にのみ使う。
保証できない場合に使うとランタイムトラップになるため、`Task { @MainActor in ... }` を選ぶ。

---

## レビューチェックリスト

Swift Concurrency を含む PR をレビューするときに確認する項目:

- [ ] アクター境界を越える型が `Sendable` に適合しているか
- [ ] `@unchecked Sendable` に、守っている不変条件とロック対象のコメントがあるか
- [ ] すでに非同期コンテキストなのに冗長な `Task { }` を作っていないか
- [ ] 長寿命 Task（無限ストリームの consume / 保持された Task）に `[weak self]` があるか
- [ ] `@MainActor` 型の `deinit` が格納プロパティ以外に触れていないか
- [ ] `deinit` 内で `Task { }` を作って `self` に触れていないか
- [ ] 分離メソッドが必要な `deinit` が `isolated deinit` または明示的 cleanup になっているか
- [ ] リアルタイム／バックグラウンドから呼ばれるコールバックが `@MainActor` 文脈の外で定義されているか
- [ ] `nonisolated(unsafe)` が上記 2 例外のいずれかで、理由コメントがあるか
- [ ] `DispatchQueue.main.async` が `@MainActor` / `MainActor.run` に置き換えられているか
- [ ] `MainActor.assumeIsolated` の使用箇所でメインスレッド保証が成立しているか
