# Swift クリーンコード規約

命名の質・関数設計・データ構造設計に関する Swift 向け規約。
Robert C. Martin の *Clean Code* 原則を Swift 向けに翻案したガイド
（[clean-code-swift](https://github.com/MaatheusGois/clean-code-swift)）を踏まえ、
「1つの関数・1つの型の内部」に閉じる可読性の観点を定める。

---

## 適用条件

`.swift` ファイルを新規作成・編集するすべての場面に適用する。
特に、命名・関数分割・データ設計のレビュー観点として用いる。

**`swift.md` との役割分担:** 本ファイルと `swift.md` は適用対象（全 `.swift` ファイル）が
重なるが、扱う観点が異なる。表記・構文・型安全性は `swift.md`、設計・可読性の質は
本ファイルに従う。

| 観点 | 参照するルール |
|------|----------------|
| 命名の表記（PascalCase / camelCase 等） | `swift.md` |
| 命名の質（語彙一貫性・検索可能性・冗長コンテキスト回避） | 本ファイル |
| 同一ロジックの重複除去（2箇所以上での関数抽出） | `swift.md`（本ファイルでは再掲しない） |
| 1関数の責務・抽象度・引数設計 | 本ファイル |
| `throws` / `Result` / `Optional` の選定 | `swift-error.md` |
| `do-catch` の `catch` 節の最低要件 | 本ファイル |
| ViewModel の `enum State` 設計 | `viewmodel.md`（本ファイルの enum 状態表現の一般則の特化版） |

**他ルールとの優先順位:** より適用範囲の狭いルール（`viewmodel.md`、`swiftui-view.md`、
`swift-error.md`、`swift-concurrency.md` 等）が対象を限定して規定している場合はそちらを
優先する。本ファイルはそれらの一般化・補足として機能する。

---

## 変数・命名

### 意味があり発音可能な名前を使う

```swift
// ❌ 悪い例
let ymdStr = DateFormatter().string(from: Date())

// ✅ 良い例
let formattedDate = DateFormatter().string(from: Date())
```

### 同じ種類の変数には同じ語彙を使う

同じ概念に対して `getUserInfo` / `fetchClientData` / `loadCustomerRecord` のように
語彙が揺れると、検索性が落ち呼び出し側の記憶負荷が増える。

```swift
// ❌ 悪い例: 同じ「取得」なのに語彙が揺れる
func getUserInfo() -> User { ... }
func fetchClientData() -> Client { ... }

// ✅ 良い例: 語彙を統一する
func fetchUser() -> User { ... }
func fetchClient() -> Client { ... }
```

### 検索可能な名前を使う（マジックナンバーの定数化）

数値・文字列リテラルを直接埋め込むと、意図が伝わらず grep でも追えない。
定数化して名前をつける。表記形式（`UPPER_SNAKE_CASE` 等）は `swift.md` に従う。

```swift
// ❌ 悪い例: 3 が何を意味するか呼び出し側から読み取れない
if retryCount > 3 { stopRetrying() }

// ✅ 良い例
let MAX_RETRY_COUNT = 3
if retryCount > MAX_RETRY_COUNT { stopRetrying() }
```

### 説明変数を使う

複雑な式の中間結果を無名のまま複数回使い回すと、各参照箇所で式を読み直す
コストが発生する。

```swift
// ❌ 悪い例: マッチ結果に毎回インデックスアクセスする
let city = (address as NSString).substring(with: match.range(at: 1))
saveCityZipCode((address as NSString).substring(with: match.range(at: 1)),
                (address as NSString).substring(with: match.range(at: 2)))

// ✅ 良い例: 一度説明変数に落とす
let city = (address as NSString).substring(with: match.range(at: 1))
let zipCode = (address as NSString).substring(with: match.range(at: 2))
saveCityZipCode(city, zipCode)
```

### メンタルマッピングを避ける

クロージャ引数やループ変数を1文字などの省略名にすると、読み手が離れた箇所から
意味を推測し直す負荷（メンタルマッピング）が生まれる。

```swift
// ❌ 悪い例
locations.forEach { l in
    dispatch(l)
}

// ✅ 良い例
locations.forEach { location in
    dispatch(location)
}
```

### 不要なコンテキストを避ける

型名をプロパティ名の prefix として繰り返すと、`user.userName` のように冗長になる。

```swift
// ❌ 悪い例
struct User {
    var userName: String
    var userAge: Int
}

// ✅ 良い例
struct User {
    var name: String
    var age: Int
}
```

---

## 関数設計

### 引数の設計（ビジネスロジック関数は最大2個）

引数が増えるほど呼び出し側の記憶負荷とテストケースの組み合わせが増える。
3個を超える場合は関連する引数をまとめた `struct` を導入する。

**適用除外:** `struct` の memberwise init、SwiftUI `View` の初期化、
Protocol 型を複数注入する DI イニシャライザ（`swift.md` / `viewmodel.md` の
DI パターンに従うもの）は対象外とする。これらは引数それぞれが独立した意味を
持ち、まとめることでかえって可読性が落ちるため。

```swift
// ❌ 悪い例: ビジネスロジック関数の引数が4個
func createMenu(title: String, body: String, buttonText: String, cancellable: Bool) {
    // ...
}

// ✅ 良い例: 関連する引数を1つの構造体にまとめる
struct MenuConfig {
    let title: String
    let body: String
    let buttonText: String
    let cancellable: Bool
}

func createMenu(config: MenuConfig) {
    // ...
}
```

### Optional 引数のデフォルト値を使う

呼び出し側で `??` による分岐を書かせるより、引数側にデフォルト値を持たせる方が
意図が明確になる。

```swift
// ❌ 悪い例
func createBrewery(name: String?) {
    let breweryName = name ?? "Default Brewery"
}

// ✅ 良い例
func createBrewery(name: String = "Default Brewery") {
    // ...
}
```

### 単一責任（1関数1責務）

```swift
// ❌ 悪い例: フィルタリングと送信が同居する
func emailClients(clients: [Client]) {
    clients.forEach { client in
        if database.lookup(client).isActive() {
            email(client)
        }
    }
}

// ✅ 良い例: 責務を分離する
func emailActiveClients(clients: [Client]) {
    clients.filter(isActiveClient).forEach(email)
}

func isActiveClient(_ client: Client) -> Bool {
    database.lookup(client).isActive()
}
```

### 関数名は何をするか明確にする

```swift
// ❌ 悪い例: 何が追加されるか名前から読めない
func addToDate(_ date: Date, _ value: Int) { ... }

// ✅ 良い例
func addingMonths(_ months: Int, to date: Date) -> Date { ... }
```

### 抽象度を1レベルに保つ

低レベルな処理（パース）と高レベルな処理（保存）が1つの関数に同居すると、
再利用性とテスト容易性が落ちる。

```swift
// ❌ 悪い例: パースと保存が同じ抽象度で混在
func parseAndSave(input: String) {
    let tokens = input.split(separator: " ")
    let record = Record(tokens: tokens.map(String.init))
    database.save(record)
}

// ✅ 良い例: 抽象度ごとに分離する
func parse(input: String) -> Record {
    Record(tokens: input.split(separator: " ").map(String.init))
}

func save(_ record: Record) {
    database.save(record)
}
```

### Boolean フラグ引数を避ける

Boolean フラグは「関数が2つ以上のことをする」サインであることが多い。
**適用範囲は自作 API に限る。** システム API（`.animation(_:value:)` 等）の
呼び出しは対象外とする。

```swift
// ❌ 悪い例
func createFile(name: String, temporary: Bool) {
    if temporary { /* ... */ } else { /* ... */ }
}

// ✅ 良い例: フラグで分岐する代わりに関数を分ける
func createTemporaryFile(name: String) { /* ... */ }
func createPermanentFile(name: String) { /* ... */ }
```

### 副作用を避ける

グローバル変数の書き換えや、参照型の引数を関数内で直接 mutate すると、
呼び出し元が意図しない変化に巻き込まれる。

```swift
// ❌ 悪い例: 参照型配列を直接 mutate し、呼び出し元にも影響する
func addItem(_ item: CartItem, to cart: inout [CartItem]) {
    cart.append(item)
}

// ✅ 良い例: 新しい配列を返す
func addingItem(_ item: CartItem, to cart: [CartItem]) -> [CartItem] {
    cart + [item]
}
```

### 既存の型への場当たり的な extension を避ける

標準型・共有型への `extension` は名前空間全体に影響するため、他モジュールとの
名前衝突リスクを生む。用途が局所的なら独立した型・関数にする。

```swift
// ❌ 悪い例: Array 全体に用途の狭いメソッドを生やす
extension Array where Element == Int {
    func diffFromBaseline(_ baseline: Int) -> [Int] { map { $0 - baseline } }
}

// ✅ 良い例: 用途を型として独立させる
enum NumberListAnalyzer {
    static func diff(_ values: [Int], from baseline: Int) -> [Int] {
        values.map { $0 - baseline }
    }
}
```

### 命令的ループより関数型スタイルを優先する

```swift
// ❌ 悪い例
var totalLines = 0
for programmer in programmers {
    totalLines += programmer.linesOfCode
}

// ✅ 良い例
let totalLines = programmers.reduce(0) { $0 + $1.linesOfCode }
```

### 条件式のカプセル化・二重否定回避

複雑な条件式は意味のある名前の関数に抽出する。`isNotX` のような二重否定を
招く命名も避ける。

```swift
// ❌ 悪い例
if fsm.state == .fetching && listNode.isEmpty {
    showSpinner()
}

// ✅ 良い例
func shouldShowSpinner(fsm: FSM, listNode: Node) -> Bool {
    fsm.state == .fetching && listNode.isEmpty
}

if shouldShowSpinner(fsm: fsm, listNode: listNode) {
    showSpinner()
}
```

### デッドコードの削除

呼ばれていない関数・型は「念のため」残さず削除する。バージョン管理システムで
いつでも復元できるため、コードベースに残す理由にならない。

---

## データ構造・オブジェクト設計

### Pure Object 的設計

共有インスタンスを直接 mutate するメソッドは、そのインスタンスを参照する
他のコードに予期しない影響を与える。可能な場合は更新後の新しいインスタンスを
返す設計を検討する。

```swift
// ❌ 悪い例: 共有インスタンスを直接書き換える
final class Account {
    var balance: Int
    func addingFunds(_ amount: Int) { balance += amount }
}

// ✅ 良い例: 新しいインスタンスを返す
struct Account {
    let balance: Int
    func addingFunds(_ amount: Int) -> Account {
        Account(balance: balance + amount)
    }
}
```

### enum による状態・種別表現

文字列比較や `switch` 文字列は typo に弱く、コンパイラのチェックも効かない。
状態・種別は `enum` で表現する。

> `viewmodel.md` の `enum State`（読み込み・成功・失敗の排他表現）は、
> この一般則を ViewModel に特化させたケースである。

```swift
// ❌ 悪い例
if status == "active" { /* ... */ }

// ✅ 良い例
enum AccountStatus {
    case active
    case suspended
    case closed
}

if status == .active { /* ... */ }
```

### computed property でのバリデーション・カプセル化

外部から無検証で直接代入できるプロパティは、不正な状態を許してしまう。

```swift
// ❌ 悪い例: 検証なしに直接代入できる
struct User {
    var age: Int
}

// ✅ 良い例: computed property でバリデーションを挟む
struct User {
    private var _age: Int

    var age: Int {
        get { _age }
        set { _age = max(0, newValue) }
    }
}
```

---

## エラーハンドリング（do-catch の最低要件）

`throws` / `Result` / `Optional` のどれを選ぶかは `swift-error.md` に従う。
本節は、いざ `do-catch` を書いた際の `catch` 節の最低要件を定める。

`catch` 節で `print(error)` だけで済ませる、あるいは何もしないことは、
エラーに対処する機会を放棄することと同義である。`swift-error.md` の `try?` と
同様に、「意図的に無視してよい」場合は理由コメントを必須とし、それ以外は
リカバリ・ユーザー通知・エラー報告のいずれかを行う。

```swift
// ❌ 悪い例: ログを流すだけで対処していない
do {
    try save(item)
} catch {
    print(error)
}

// ✅ 良い例: 状態に反映し、ユーザーに伝える
do {
    try save(item)
} catch {
    state = .error(error)
}

// ✅ 良い例: 無視してよい場合は理由コメント必須
// バックアップが既に存在しない場合は無視してよい
try? FileManager.default.removeItem(at: backupURL)
```

---

## コメント

### コメントアウトされたコードを残さない

バージョン管理システムで過去のコードはいつでも復元できる。コメントアウトされた
コードはノイズにしかならないため削除する。

### `MARK:` を構造分割の代替にしない

`MARK:` は Xcode のジャンプバーで広く使われる Swift の慣習であり、
**同一型内で関連するメンバーを探すための用途は許容する。**
禁止するのは、複数の責務を `MARK:` で区切っただけで1つのファイル・1つの型に
同居させ続け、型・ファイルの分割を先延ばしにすることである。

```swift
// ❌ 悪い例: MARK: で責務を区切っただけで、
//          ネットワーキングと永続化が同じ型に同居し続けている
final class UserService {
    // MARK: - Networking
    func fetchUser() { /* ... */ }

    // MARK: - Persistence
    func saveUser() { /* ... */ }
}

// ✅ 良い例: 責務ごとに型を分割する。MARK: は同一型内のメンバー探索用途に限る
final class UserRepository {
    // MARK: - CRUD
    func fetchUser() { /* ... */ }
    func saveUser() { /* ... */ }
}
```

---

## レビューチェックリスト

- [ ] 変数名が意味を持ち発音可能か（省略形・連続子音の羅列がないか）
- [ ] 同じ概念に対する関数名・変数名がプロジェクト内で揺れていないか
- [ ] マジックナンバーが定数化され、検索可能な名前になっているか
- [ ] 複雑な式（正規表現マッチ結果等）を説明変数に落としているか
- [ ] クロージャ引数名が1文字等の省略名になっていないか
- [ ] 型名がプロパティ名の冗長な prefix になっていないか（`User.userName` 等）
- [ ] ビジネスロジック関数の引数が3個以上の場合、struct への集約を検討したか
      （struct memberwise init・SwiftUI View・Protocol型 DI init は対象外）
- [ ] Optional 引数にデフォルト値を与え、呼び出し側の `??` 分岐を避けているか
- [ ] 1つの関数が複数の責務を持っていないか
- [ ] 1つの関数内で抽象度の異なる処理（低レベル処理と高レベル処理）が混在していないか
- [ ] 自作関数の引数に Boolean フラグを使い、内部で分岐していないか
- [ ] グローバル変数の書き換え・参照型引数の直接 mutate による副作用がないか
- [ ] 既存の型への場当たり的な extension で名前衝突リスクを作っていないか
- [ ] map/filter/reduce で書けるループを命令的に書いていないか
- [ ] 複雑な条件式が意味のある名前の関数にカプセル化されているか
- [ ] `isNotX` のような二重否定を招く命名がないか
- [ ] 呼ばれていない関数（デッドコード）が残っていないか
- [ ] 状態や種別を文字列比較・switch 文字列ではなく enum で表現しているか
- [ ] 可変性を持つプロパティが computed property 等で検証・カプセル化されているか
- [ ] do-catch の catch 節が何もしない場合、理由コメントとリカバリ/通知/報告のいずれかがあるか
- [ ] コメントアウトされたコードが残っていないか
- [ ] `MARK:` が複数責務を1ファイルに同居させる言い訳になっていないか
