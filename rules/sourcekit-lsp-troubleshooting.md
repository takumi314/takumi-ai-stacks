# SourceKit-LSP 診断の扱い規約

エディタ上の診断表示（赤線）と実際のビルド結果が食い違う場合に、
どちらを信じ、何を実行してから判断するかを定める。

---

## 適用条件

- SPM（`Package.swift`）ベースのプロジェクトで、SourceKit-LSP による診断を使う場面
  （VS Code / Neovim 等のエディタ表示）

Xcode（`.xcodeproj`）のビルドエラー、および `swift build` が出力するエラーには適用しない。
それらは実エラーとして通常どおり修正する。

---

## 判断の原則

`swift build` / `swift test` の実行結果を唯一の真実（Single Source of Truth）とする。
エディタの診断表示は、実行結果で裏付けが取れるまで判断材料にしない。

診断を見つけたら、次の順で処理する。

1. 該当ターゲットに対し `swift build`（テストターゲットなら `swift test`）を実行する
2. **通った** → LSP の表示として扱い、コードは変更しない。「ビルドは通っている」とユーザーへ報告する
3. **通らなかった** → 実エラーとして修正する

**診断表示だけを根拠にコードを変更してはならない。** 必ず 1 を実行してから判断する。
表示を消すことではなく、ビルドが通る状態を保つことが目的である。

---

## 実エラーとして扱うもの

### `ObservableObject` の macOS availability 診断

`Package.swift` に `platforms:` の指定がない場合、macOS のデプロイメントターゲットは
既定値まで下がり、`ObservableObject`（Combine / macOS 10.15+）は **実エラー**になる。
LSP の誤表示ではないため、無視してはならない。

判断手順:

1. `Package.swift` に `platforms:` があるか確認する
2. ない、または macOS が 10.15 未満 → `platforms` を追加して修正する

   ```swift
   platforms: [.iOS(.v13), .macOS(.v10_15)]
   ```

3. すでに適切に指定されているのに表示が残る → 「対応手順」節へ進む

---

## LSP 表示の可能性があるもの

### `No such module 'Testing'`

テストターゲット内で発生し、かつ `swift test` が通る場合は LSP のパス解決による表示である。
ただし以下の実エラー経路があるため、**`swift test` の実行を省略してはならない**。

- ツールチェーンが Swift 5.x（Swift Testing は Swift 6 / Xcode 16 以降が必須）
- `swift-tools-version` が古い
- `xcode-select` が Command Line Tools を指しており、想定と別のツールチェーンが使われている

判断手順:

1. `swift test` を実行する。通れば LSP 表示として扱い、コードは変更しない
2. 通らなければ `swift --version` と `xcode-select -p` を確認し、ツールチェーンを是正する

---

## 対応手順

ビルドが通るのに診断が残り続ける場合、以下の順に試す（またはユーザーに提案する）。

1. `rm -rf .build` を実行し、LSP を再起動する
2. 1 で解消しない場合のみ `.swiftpm/xcode` の削除を検討する

   > ⚠️ `.swiftpm/xcode` には Xcode のスキーム等の共有設定が含まれる。削除すると失われるため、
   > 実行前に必ずユーザーの承認を得る（グローバル CLAUDE.md「禁止事項」の対象）。

3. ツールチェーンを確認する（`swift --version` / `xcode-select -p`）

---

## 禁止事項

| ❌ 禁止 | ✅ 代わりに |
|------|----------|
| LSP のエラーを消すためだけに `#if canImport(Testing)` や `#if os(iOS)` のガードを追加する | `swift build` / `swift test` で実エラーか確認し、実エラーなら原因（`platforms` 指定・ツールチェーン）を直す |
| LSP の警告回避のために `ObservableObject` を `@Observable`（Observation / iOS 17+ / macOS 14+）へ移行する | `Package.swift` の `platforms` を適切に指定する。`@Observable` への移行は最小サポート OS を引き上げる設計判断であり、警告回避を理由に行わない |
| ビルドを実行せずに診断表示だけを根拠にコードを修正する | 必ず `swift build` / `swift test` を実行してから判断する |
| ユーザーの承認なく `.swiftpm/xcode` を削除する | `.build` の削除を先に試し、`.swiftpm/xcode` は影響を説明して承認を得る |

---

## レビューチェックリスト

- [ ] 診断を根拠にした変更に、`swift build` / `swift test` の実行結果が伴っているか
- [ ] `#if canImport` / `#if os` のガードが、LSP 回避目的で追加されていないか
- [ ] `ObservableObject` の availability 診断に対し、`platforms` 指定で対処しているか
- [ ] `@Observable` への移行が、最小サポート OS の設計判断として行われているか
- [ ] `.swiftpm/xcode` の削除がユーザー承認を経ているか
