# swift-syntax 実装規約

swift-syntax / SwiftParser を使うコードの実装規約。
`swift.md` と `swift-error.md` の規約に加え、本ファイルの規約も満たすこと。

---

## 適用条件

以下のいずれかが含まれるソースに適用する:

- `import SwiftSyntax` / `import SwiftParser` / `import SwiftSyntaxBuilder`
- `SyntaxVisitor` / `SyntaxRewriter` 派生型
- `SourceLocationConverter` / `TriviaPiece` / `AbsolutePosition` の直接利用

---

## 名前衝突の回避

独自 `SourceLocation` 型と `SwiftSyntax.SourceLocation` が同名衝突するときは、ファイル冒頭で `typealias` により別名化する。

```swift
private typealias SyntaxLocation = SwiftSyntax.SourceLocation
private typealias CoreLocation = MyModule.SourceLocation
```

- ファイルスコープが原則。モジュールスコープに置くと別ファイルでの取り違えを誘発する
- import 順序に依存して意味が変わるコードは禁止

---

## Sendable 境界

- `SyntaxVisitor` / `SyntaxRewriter` 派生 class は **mutable state を持つため非 Sendable**
- 公開ロール（`SafeNodeExtractor` 等）を `Sendable` で公開する場合は **`struct` + `private final class _Visitor` の委譲パターン**を使う

```swift
public struct MyExtractor: SafeNodeExtractor {
    public func extract(from tree: SourceFileSyntax) -> [SafeNode] {
        let visitor = _Visitor()
        visitor.walk(tree)
        return visitor.nodes
    }
}

private final class _Visitor: SyntaxVisitor {
    var nodes: [SafeNode] = []
    ...
}
```

- `SyntaxVisitor` / `SyntaxRewriter` インスタンスは **関数スコープで都度生成・破棄**する
- ストアドプロパティとして保持しない（タスクをまたいだ共有で競合が起きる）

---

## Serializer のインデックス整合性検査（位置）

ソースを構文木 → 編集適用 → 文字列に戻す Serializer は、まず編集の **位置（offset / range）** が
パス間・エンコーディング間でずれないことを保証する。位置がずれた状態ではどんな「内容検査」も意味をなさない。
本節は「位置」、次節は「内容」を扱う直交した観点。

### 汎用原則

- **起点・終点のセマンティクスを 1 つに固定する**
  - UTF-8 byte offset / UTF-16 code unit / Unicode scalar / `String.Index` 系が言語・API で混在しうる
  - 代表例: Swift Stdlib `replaceSubrange` は `String.Index`、LSP `TextDocumentEdit` は UTF-16、
    libcst は scalar、ts-morph は code unit — 同じ Serializer 内で混ぜない
- **抽出パスと適用パスで同じ算出ロジック**を使う
  - 片方が「終端 trivia を含む位置」、片方が「含まない位置」になると置換が誤位置に滑り込む
- **半開区間 `[start, end)` か 閉区間 `[start, end]` かを明示**する
  - API ごとに慣習が異なる。doc-comment に明記して呼び出し側の取り違えを防ぐ
- **複数の置換を適用する順序を固定**する
  - 同一ソースに昇順で破壊的置換を当てると、先行置換で後続 offset がずれる
  - **降順（末尾から先頭へ）に適用**するか、エディタ系の Edit 配列 API（LSP `TextDocumentEdit.edits` 等、
    全 edit を「適用前のソース」基準の offset で受け取る API）に任せる
- **line / column の 0-based / 1-based を doc-comment に明記**する

### Swift / swift-syntax の場合

Trivia（コメント・空白）は隣接 token 間で一意所属する。改行までは前 token の `trailingTrivia`、改行以降は次 token の `leadingTrivia`。

| 用途 | 起点 |
|------|------|
| leading trivia の起点 | `token.position.utf8Offset` |
| trailing trivia の起点 | `token.endPositionBeforeTrailingTrivia.utf8Offset` |
| 各 piece の長さ | `TriviaPiece.sourceLength.utf8Length`（`.text.count` ではない） |

- `token.endPosition` は trailing trivia を含む末尾 → 起点に使うと off-by-one を生む
- `token.positionAfterSkippingLeadingTrivia` は **token 本体**の先頭 → trivia を含めたい場合は使わない
- 第1パス（抽出）と第2パス（置換）で **同じ offset 算出ロジック**を使うこと
  - 片側で `endPosition.utf8Offset` を使うと整合が崩壊し、置換が誤位置に適用される
- `SourceLocationConverter` の `line` / `column` は **1-based**、column は **UTF-8 バイト位置**

---

## Serializer の置換テキスト検査（内容）

位置が正しくても、**置換テキスト自体**が対象ノードの境界トークン（終端クォート / コメント終端 / 区切り文字）を含むと、
parse → serialize → re-parse で構文が壊れる。位置整合とは独立した観点として扱う。

> ここでの「構文非破壊」は「parse → serialize → re-parse が `hasError == false` で完結する」を意味する。
> バイト恒等は無改変ケースの保証であり、置換ケースでは「再 parse 可能であること」を最低限の保証ラインとする。

### 汎用原則

- 置換適用前に、対象ノード種別ごとの **終端トークンを含まないこと** を検査する
- 違反時は **置換スキップ + 警告通知**を基本とする（黙って escape を挿入しない）
- 警告は注入された Sink（`WarningSink` 等）に流す。**ライブラリ層の stderr 直書きは禁止**
  - 参照: `~/.claude/skills/code-review/guidelines/common.md` 「ライブラリ層の I/O 副作用」
- 代表対象 API: Swift `replaceSubrange` / swift-syntax の Rewriter、LSP `TextDocumentEdit`、
  libcst の `with_changes`、ts-morph の `replaceWithText` — いずれも「位置 OK でも内容 NG」が独立に発生する

### Swift / swift-syntax の場合

文字列リテラル（`StringLiteralExprSyntax.segments`）とコメント（`TriviaPiece`）に対し、以下を満たすこと。

| ノード種別 | 検査項目 | 違反時の挙動 |
| --- | --- | --- |
| 通常文字列 `"..."` | 置換テキストに `"`（ベアクォート）または改行を含まない | 置換をスキップ |
| 複数行文字列 `"""..."""` | 置換テキストに `"""`（連続 3 個以上のダブルクォート）を含まない | 置換をスキップ |
| Raw 文字列 `#"..."#`（N 個の `#`） | 置換テキストに `"` + N 個以上の連続 `#` を含まない | 置換をスキップ |
| Raw 複数行 `#"""..."""#` | 上記 2 つを併用（`"""` + N 個以上の `#` も） | 置換をスキップ |
| 行コメント `// ...` | 置換テキストに改行（`\n` / `\r`）を含まない | 置換をスキップ |
| ブロックコメント `/* ... */` | 置換テキストに `*/` を含まない | 置換をスキップ |
| doc コメント `///` / `/** */` | 上記コメント検査と同じ | 置換をスキップ |

ノード種別の判定:

```swift
// N（ポンド数）の取得
let poundCount = node.openingPounds?.text.count ?? 0
// 複数行判定
let isMultiline = node.openingQuote.tokenKind == .multilineStringQuote
```

コメント trivia の検査例:

```swift
// 行コメントの検査（CRLF は 1 grapheme のため contains("\n") では検出不可 → スカラー単位で照合）
guard !replacement.unicodeScalars.contains(where: { $0 == "\n" || $0 == "\r" }) else { skip }

// ブロックコメントの検査
guard !replacement.contains("*/") else { skip }
```

---

## SafeNode.kind と offset の整合

置換用辞書 `[utf8Offset: 置換テキスト]` だけで分岐すると、kind 不一致の置換テキストが
誤って別種ノードに流れ込み構文を壊しうる（例: コメント用テキストが文字列セグメントに注入される）。

- 置換辞書のバリューには **kind 情報を含めて持つ**こと（`[Int: (NodeKind, String)]` または `[Int: SafeNode]`）
- 各 visit 側で `kind == 期待される種別` を確認してから書き換える

---

## Dictionary 構築（公開 API 入口）

- 置換用の `[utf8Offset: String]` 辞書を **公開関数の引数から構築**する場合、
  `Dictionary(uniqueKeysWithValues:)` は使わない（同一オフセット重複で `fatalError`）
- 詳細は `~/.claude/skills/code-review/guidelines/swift.md` の
  「`Dictionary(uniqueKeysWithValues:)` の同一キー混入」を参照

---

## エラーリカバリと hasError

- `SwiftParser.Parser.parse(source:)` は構文エラー時にもリカバリして `SourceFileSyntax` を返す（throw しない）
- リカバリ後の木に対する抽出・置換は不完全になりうる → これを doc-comment に明示すること
- 抽出側 API が `throws` を持つ場合、それは Parser プロトコル要求由来か実体由来かを doc-comment で区別する

---

## ラウンドトリップテスト

置換を行うコードには以下のテストを必ず用意する。

1. **無改変ケース**: `parse → serialize` がバイト恒等（`result == source`）
2. **置換ケース**: 期待出力との一致 + `try Parser.parse(source: result); #expect(!tree.hasError)`
3. **構文破壊ケース**: 通常文字列に `"`、複数行に `"""`、Raw に `"#`、コメントに改行 / `*/` を含む置換が
   スキップされ、結果が元ソースのままになることを確認

無改変バイト恒等チェックだけでは「置換後の構文が valid か」を保証できないため、
**再 parse + `hasError` 検証**を共通ヘルパーにする:

```swift
private func assertReparseable(_ source: String) throws {
    let tree = try SwiftSourceParser().parse(source: source, path: FilePath("rt.swift"))
    #expect(!tree.hasError, "serialize 結果が再 parse で構文エラー")
}
```

---

## レビューチェックリスト

- [ ] 名前衝突の `typealias` が冒頭にある
- [ ] `Sendable` 境界が `struct` + `private final class _Visitor` で表現されている
- [ ] `SyntaxVisitor` / `SyntaxRewriter` 派生がストアドプロパティに漏れていない
- [ ] 第1パスと第2パスで同じ offset 算出ロジックを使っている
- [ ] offset の単位（UTF-8 byte / UTF-16 / scalar / `String.Index`）と半開/閉区間が doc-comment に明示されている
- [ ] 複数置換の適用順序が降順または Edit 配列 API で固定され、後続 offset のズレを防いでいる
- [ ] 置換テキストの構文非破壊検査が「通常 / 複数行 / Raw / Raw 複数行」すべてで実装されている
- [ ] コメント置換に対する改行 / `*/` 検査がある
- [ ] 違反時の警告が `WarningSink` 経由で、stderr 直書きでない
- [ ] 置換辞書のバリューに `NodeKind` が含まれており、kind 不一致の流れ込みを防いでいる
- [ ] `Dictionary(uniqueKeysWithValues:)` を公開 API 入口で使っていない
- [ ] `hasError` 後の挙動が doc-comment に書かれている
- [ ] ラウンドトリップテストに「再 parse → `hasError == false`」が含まれている
