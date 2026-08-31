# ViewModel 実装規約

MVVM 構成における ViewModel の実装規約。

---

## 適用条件

View（SwiftUI / UIKit）に対応する ViewModel を新規作成・変更する場面に適用する。

**他ルールとの優先順位:** 並行性の詳細は `swift-concurrency.md`、
エラー表現の選択は `swift-error.md`、テストは `swift-test.md` を優先する。

---

## ViewModel の責務

- View からのユーザー入力を受け取る
- ビジネスロジックの実行
- UI に必要なデータを管理・提供する
- View の更新をトリガーする

View に属する描画・レイアウトの判断は ViewModel に置かない（`swiftui-view.md`）。

---

## 基本規則

- `@MainActor` で修飾する
- `final class` + `ObservableObject` で宣言する
- `@Published` を付けるのは、View が描画に使うプロパティに限る。
  View から参照されないプロパティに `@Published` を付けない
- View から参照されないプロパティには `private` を付ける

---

## 状態管理

読み込み・成功・失敗のように **同時に成立しない状態** は、複数の Boolean ではなく
`enum State` 1 つで表現する。

```swift
@MainActor
final class ItemListViewModel: ObservableObject {
    @Published private(set) var state: State = .idle

    enum State {
        case idle
        case loading
        case success([Item])
        case error(Error)
    }
}
```

`@Published var isLoading = false` と `@Published var hasError = false` を併置しない。
両方が `true` になる、あるいは両方が `false` のまま止まるといった、
表現できてしまう不正な組み合わせが生まれるため。

---

## 非同期処理

- `async/await` を使用する（詳細は `swift-concurrency.md`）
- 進行中の処理を中断する必要がある場合、`Task` をプロパティに保持し `task?.cancel()` で中断する
- 新しい処理を開始する前に、前回の `Task` をキャンセルする

```swift
private var loadTask: Task<Void, Never>?

func load() {
    loadTask?.cancel()
    loadTask = Task { [weak self] in
        // ...
    }
}
```

---

## Dependency Injection

外部依存は Protocol 経由で受け取り、イニシャライザで注入する。
テスト時に Mock を差し替えられる形にする（`swift-test.md`）。

```swift
init(repository: MyRepositoryProtocol = DefaultRepository()) {
    self.repository = repository
}
```

---

## エラーハンドリング

エラーは `State` の中で表現し、**`State` と独立したエラー用プロパティを併置しない。**
`state` と `errorMessage` の両方を持つと、`state` が `.success` なのに
`errorMessage` が残るといった不整合が生じるため。

View へ表示文字列が必要な場合は、`State` から導出する。

```swift
var errorMessage: String? {
    guard case .error(let error) = state else { return nil }
    return error.localizedDescription
}

/// エラー表示を閉じたときに呼ぶ。
func clearError() {
    guard case .error = state else { return }
    state = .idle
}
```

`State` を持たない単純な ViewModel に限り、`@Published var errorMessage: String?` を
単独で使ってよい。その場合も操作後に `clearError()` でリセットする。

---

## レビューチェックリスト

- [ ] `@MainActor` + `final class` + `ObservableObject` で宣言されているか
- [ ] `@Published` が View の描画に使うプロパティだけに付いているか
- [ ] View から参照されないプロパティに `private` が付いているか
- [ ] 同時に成立しない状態が複数の Boolean ではなく `enum State` で表現されているか
- [ ] `isLoading` と `hasError` のような Boolean の併置が残っていないか
- [ ] `State` とは別のエラー用プロパティを併置していないか（導出プロパティになっているか）
- [ ] 中断が必要な非同期処理で `Task` を保持し、再実行前にキャンセルしているか
- [ ] 外部依存が Protocol 経由でイニシャライザ注入されているか
- [ ] 描画・レイアウトの判断が ViewModel に混入していないか
