# Swift オブジェクト指向設計・SOLID規約

型と型の関係（継承・合成・依存）に関する Swift 向け設計規約。
Robert C. Martin の *Clean Code* / SOLID 原則を Swift 向けに翻案したガイド
（[clean-code-swift](https://github.com/MaatheusGois/clean-code-swift)）を踏まえ、
「1つの関数の中身」ではなく「型同士の関係」のレベルの設計判断を定める。

---

## 適用条件

`class` / `struct` / `protocol` を新規設計する場面、および既存の型階層に
変更を加える場面に適用する。関数内部の設計（引数数・単一責任・抽象度等）は
`swift-clean-code.md` に従う。

**他ルールとの優先順位:** ViewModel / View 固有の型設計は `viewmodel.md` /
`swiftui-view.md` を優先する。外部依存の Protocol 経由 DI は `swift.md` と
`viewmodel.md` に既に規定があり、本ファイルの DIP はその設計原理の説明を
補足するのみで、実装上の新たな規約は追加しない。

---

## 継承よりコンポジションを優先する

継承は "is-a"（〜は〜の一種である）の関係にのみ使う。実質的には "has-a"
（〜は〜を持つ）の関係を継承で表現すると、無関係な振る舞いまで引き継いで
しまう。

**適用除外:** `UIViewController` / `NSManagedObject` / `XCTestCase` など
フレームワークが継承を要求する型、および `ObservableObject` のように
`class` 適合が前提となるプロトコルへの適合は対象外とする。

```swift
// ❌ 悪い例: EmployeeTaxData は Employee の一種ではなく、
//          Employee が持つデータである（"has-a" を継承で表現）
class Employee {
    var name: String
    init(name: String) { self.name = name }
}

class EmployeeTaxData: Employee {
    var ssn: String
    init(name: String, ssn: String) {
        self.ssn = ssn
        super.init(name: name)
    }
}

// ✅ 良い例: 合成で "has-a" を表現する
struct EmployeeTaxData {
    let ssn: String
}

struct Employee {
    let name: String
    var taxData: EmployeeTaxData?
}
```

---

## 条件分岐よりポリモーフィズムを検討する

`as?` による型チェック分岐が連鎖すると、種別が増えるたびに分岐箇所を
すべて修正する必要が生まれる。プロトコル適合による統一 API に置き換える。

```swift
// ❌ 悪い例: 型ごとに面積計算のロジックを分岐する
func area(of shape: Any) -> Double {
    if let rect = shape as? Rectangle {
        return rect.width * rect.height
    } else if let circle = shape as? Circle {
        return circle.radius * circle.radius * .pi
    }
    return 0
}

// ✅ 良い例: プロトコルで統一 API にする
protocol Shape {
    var area: Double { get }
}

struct Rectangle: Shape {
    let width: Double
    let height: Double
    var area: Double { width * height }
}

struct Circle: Shape {
    let radius: Double
    var area: Double { radius * radius * .pi }
}
```

---

## SOLID原則

各原則を Swift の言語機能（`protocol` / `enum` / `extension`）でどう体現するかを示す。

### 単一責任の原則（SRP）

1つの型が複数の変更理由を持つと、一部の変更が無関係な機能に影響を及ぼす
リスクが生まれる。

> `swift.md` の「1ファイル1型定義」「同一ロジックの重複除去」とは粒度が異なる
> 補足であり、こちらは「1つの型が持つ責務の数」に焦点を当てる。

```swift
// ❌ 悪い例: 設定変更と認証確認が同じ型に同居する
final class UserSettings {
    func changeSettings(_ settings: Settings) {
        if verifyCredentials() { /* ... */ }
    }
    func verifyCredentials() -> Bool { /* ... */ }
}

// ✅ 良い例: 責務ごとに型を分ける
struct UserAuthenticator {
    func verifyCredentials() -> Bool { /* ... */ }
}

final class UserSettings {
    let auth: UserAuthenticator
    func changeSettings(_ settings: Settings) {
        if auth.verifyCredentials() { /* ... */ }
    }
}
```

### 開放閉鎖の原則（OCP）

種別が増えるたびに既存の `if` / `switch` を書き換える設計は、変更のたびに
既存の動作確認済みコードへ手を入れるリスクを生む。`protocol` で拡張可能にし、
種別追加は型を増やすだけで済むようにする。

```swift
// ❌ 悪い例: 割引種別を追加するたびに switch を修正する
func discount(for type: String, price: Double) -> Double {
    switch type {
    case "member": return price * 0.9
    case "vip": return price * 0.8
    default: return price
    }
}

// ✅ 良い例: 新しい割引ポリシーは型を追加するだけで拡張できる
protocol DiscountPolicy {
    func apply(to price: Double) -> Double
}

struct MemberDiscount: DiscountPolicy {
    func apply(to price: Double) -> Double { price * 0.9 }
}

struct VIPDiscount: DiscountPolicy {
    func apply(to price: Double) -> Double { price * 0.8 }
}
```

### リスコフの置換原則（LSP）

サブタイプは、親の契約（呼び出し可能なメソッドが正常に動作すること）を
破ってはならない。呼べば必ず `fatalError` するメソッドを持つサブタイプは
契約違反の兆候である。

```swift
// ❌ 悪い例: Penguin は fly() を呼ぶと必ず落ちる
protocol Bird {
    func fly()
}

struct Penguin: Bird {
    func fly() { fatalError("Penguins can't fly") }
}

// ✅ 良い例: 飛べる種のみが適合するプロトコルに分ける
protocol FlyingBird {
    func fly()
}

struct Sparrow: FlyingBird {
    func fly() { /* ... */ }
}

struct Penguin {
    func swim() { /* ... */ }
}
```

### インターフェース分離の原則（ISP）

1つのプロトコルに無関係な要件を詰め込むと、適合する型が使わないメンバーの
実装を強制される。役割ごとにプロトコルを分離する。

```swift
// ❌ 悪い例: RobotWorker は eat() を空実装するしかない
protocol Worker {
    func work()
    func eat()
}

// ✅ 良い例: 役割ごとにプロトコルを分離する
protocol Workable {
    func work()
}

protocol Eatable {
    func eat()
}

struct RobotWorker: Workable {
    func work() { /* ... */ }
}
```

### 依存性逆転の原則（DIP）

高レベルモジュールは具象型ではなく抽象（`protocol`）に依存する。

> `viewmodel.md` の「外部依存は Protocol 経由でイニシャライザ注入する」は、
> 既に DIP の実践例である。本節はその設計原理の説明であり、実装上の規約は
> `viewmodel.md` / `swift.md` に一本化する。

```swift
// ❌ 悪い例: 具象型に直接依存する
final class ReportGenerator {
    let database = MySQLDatabase()
}

// ✅ 良い例: Protocol 経由で依存する
protocol DatabaseProtocol {
    func fetchRecords() -> [Record]
}

final class ReportGenerator {
    let database: DatabaseProtocol
    init(database: DatabaseProtocol) {
        self.database = database
    }
}
```

---

## レビューチェックリスト

- [ ] "is-a" ではなく実質 "has-a" の関係を継承で表現していないか
      （フレームワークが要求する継承・class 適合必須プロトコルは対象外）
- [ ] `as?` の型チェック分岐が連鎖しており、プロトコル適合で置き換えられないか
- [ ] 1つの型が複数の責務（変更理由）を抱えていないか
- [ ] 種別追加のたびに既存の `if` / `switch` を変更する設計になっていないか
- [ ] サブタイプが親の契約を破っていないか（呼べば必ず失敗するメソッド等）
- [ ] プロトコルが適合先にとって不要なメンバーを含んでいないか
- [ ] 具象型に直接依存せず Protocol 経由で依存しているか
