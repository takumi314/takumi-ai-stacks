# SwiftUI View 実装規約

## 適用条件

SwiftUI の `View` を新規作成・変更する場面に適用する。

**他ルールとの優先順位:** 状態管理とビジネスロジックは `viewmodel.md`、
並行性は `swift-concurrency.md`、一般規約は `swift.md` に従う。

## UI フレームワークの選択
**iOS開発では SwiftUI を優先する。** UIKit は SwiftUI で実現できない機能・既存 UIKit コードとの互換性が必要な場合のみ使用する。

## コンポーネント化
View を小さな部品に分割して再利用性を高める。各 View は1つの責務を持つ（レイアウト・スタイリング・ロジックを分離する）。

## Color 指定
- Asset Catalog（`Assets.xcassets`）で Color Set を定義して使用する
- Light/Dark モード対応・アクセシビリティカラーも定義する
- 直接 RGB 値（`Color(red:green:blue:)`）は使わない

## Font 指定
システムフォント + Text Style を使用する。

使用可能な Text Styles：`.largeTitle` / `.title` / `.title2` / `.title3` / `.headline` / `.subheadline` / `.body` / `.callout` / `.caption` / `.caption2`

## State 管理
- `@State` は `private` で定義する
- アプリ全体の状態は `@EnvironmentObject` で共有する
- `@FocusState` をコンポーネント階層をまたいで共有する場合は、最上位の親 View に定義し `FocusState<Bool>.Binding` を子へ渡すこと。子が `@FocusState` を保持すると、親からの SwiftUI ネイティブなフォーカス操作ができなくなり UIKit 依存を招く。

## その他
- Modifier は宣言的に、読みやすい順序で適用
- View は再利用可能な単位に分割
- レイアウトは `VStack`, `HStack`, `ZStack` で構築
- `GeometryReader` は最小限の使用
- `#Preview` マクロで UI を確認する

---

## レビューチェックリスト

- [ ] UIKit ではなく SwiftUI で実装されているか（UIKit なら理由が明示されているか）
- [ ] View が 1 つの責務に収まる単位に分割されているか
- [ ] Color が Asset Catalog の Color Set 経由で指定されているか
- [ ] `Color(red:green:blue:)` による直接指定が残っていないか
- [ ] Light / Dark モードとアクセシビリティカラーが定義されているか
- [ ] Font がシステムフォント + Text Style で指定されているか
- [ ] `@State` が `private` で定義されているか
- [ ] アプリ全体で共有する状態が `@EnvironmentObject` になっているか
- [ ] 階層をまたぐ `@FocusState` が最上位の親に定義され、子へ `Binding` で渡されているか
- [ ] `GeometryReader` の使用が必要最小限か
- [ ] `#Preview` が定義されているか
