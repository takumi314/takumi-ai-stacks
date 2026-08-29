# Conventional Commits 規約

## フォーマット

```
<type>(<scope>): <subject>

[body]

[footer]
```

- **type**: 変更の種類（feat, fix, style等）
- **scope**: 変更の影響範囲（オプション、原則省略）
- **subject**: 50文字以内の簡潔な説明
- **body**: 詳細説明（必要な場合のみ）
- **footer**: Breaking Changes等の記載（必要な場合のみ）

## typeの選択基準

| type | 使う場面  |
| ------ | --------- |
| feat | 新機能の追加 |
| fix | バグの修正パッチ |
| style | 画面レイアウトの変更 |
| docs | ドキュメントの変更 |
| ref | リファクタリング |
| perf | パフォーマンスの改善や向上 |
| test | テストの追加・修正 |
| chore | ビルド設定、依存関係の更新、その他 |

## scopeの選択基準

**原則：scope は省略する**

複数のファイルに影響する場合のみ、以下から該当する scope を選択してください。

| scope | 説明  |
| ------ | --------- |
| ui | UI コンポーネント、ビュー、画面レイアウト |
| animation | アニメーション処理、Transition、Timer |
| app | アプリケーション全体の設定、エントリーポイント |
| img | 画像、アイコン、アセット |
| test | テストコード |
| resource | リソースファイル、設定ファイル |
| config | プロジェクト設定、ビルド設定 |
| none | 複数領域に影響するが、特定の scope に該当しない場合（scope を省略） |

## Breaking Changes（破壊的変更）

変更内容に破壊的な変更が含まれる場合、`<type>(<scope>)` または `<type>` の後に `!` を付け加えます。

**例:**
```
feat(api)!: API レスポンス形式を変更
```

`!` は、その変更が既存の互換性を損なうことを示します。
