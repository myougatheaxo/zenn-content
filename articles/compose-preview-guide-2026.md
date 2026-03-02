---
title: "Compose Preview完全ガイド — @Previewを最大限活用する"
emoji: "👁️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "preview"]
published: true
---

## この記事で学べること

`@Preview`を使いこなせば、**エミュレータ不要**でUIを確認できます。開発速度を劇的に上げるPreviewの活用法を解説します。

---

## 基本の@Preview

```kotlin
@Preview(showBackground = true)
@Composable
fun GreetingPreview() {
    MyAppTheme {
        Text("Hello, World!", style = MaterialTheme.typography.titleLarge)
    }
}
```

`showBackground = true`で白背景がつきます。

---

## 複数Previewの並列表示

```kotlin
@Preview(name = "Light", showBackground = true)
@Preview(name = "Dark", showBackground = true, uiMode = Configuration.UI_MODE_NIGHT_YES)
@Composable
fun CardPreview() {
    MyAppTheme {
        ItemCard(item = Item("サンプル", "説明テキスト"))
    }
}
```

ライト/ダーク両方を同時プレビュー。

---

## デバイスサイズのプレビュー

```kotlin
@Preview(name = "Phone", device = Devices.PIXEL_7)
@Preview(name = "Tablet", device = Devices.PIXEL_TABLET)
@Preview(name = "Foldable", device = Devices.PIXEL_FOLD)
@Composable
fun ResponsivePreview() {
    MyAppTheme {
        HomeScreen()
    }
}
```

---

## フォントサイズ変更

```kotlin
@Preview(name = "Normal", fontScale = 1.0f)
@Preview(name = "Large", fontScale = 1.5f)
@Preview(name = "Largest", fontScale = 2.0f)
@Composable
fun AccessibilityPreview() {
    MyAppTheme {
        SettingsItem(title = "通知設定", subtitle = "プッシュ通知のオン/オフ")
    }
}
```

フォントサイズの拡大テストでアクセシビリティを確認。

---

## PreviewParameterProvider

```kotlin
class UserPreviewProvider : PreviewParameterProvider<User> {
    override val values = sequenceOf(
        User("田中太郎", "tanaka@example.com", true),
        User("佐藤花子", "sato@example.com", false),
        User("", "", false)  // 空データのケース
    )
}

@Preview(showBackground = true)
@Composable
fun UserCardPreview(
    @PreviewParameter(UserPreviewProvider::class) user: User
) {
    MyAppTheme {
        UserCard(user)
    }
}
```

複数のデータパターンを**一度にプレビュー**。エッジケースも確認できます。

---

## カスタムアノテーション

```kotlin
@Preview(name = "Light", showBackground = true)
@Preview(name = "Dark", uiMode = Configuration.UI_MODE_NIGHT_YES)
@Preview(name = "Large Font", fontScale = 1.5f)
annotation class ComponentPreviews

@ComponentPreviews
@Composable
fun ButtonPreview() {
    MyAppTheme {
        Button(onClick = {}) { Text("送信") }
    }
}
```

共通のPreview設定を**アノテーション化**すれば、毎回書く手間が省けます。

---

## Interactiveモード

Android Studio上で`@Preview`の右上にある再生ボタンを押すと、**インタラクティブモード**でクリックやスクロールを試せます。

```kotlin
@Preview(showBackground = true)
@Composable
fun InteractiveCounterPreview() {
    MyAppTheme {
        var count by remember { mutableStateOf(0) }
        Button(onClick = { count++ }) {
            Text("Count: $count")
        }
    }
}
```

---

## Previewのベストプラクティス

| 項目 | 推奨 |
|------|------|
| テーマ | 必ず`MyAppTheme`で囲む |
| データ | サンプルデータを直接渡す |
| サイズ | `widthDp`/`heightDp`で固定 |
| 命名 | `〇〇Preview`で統一 |
| 配置 | 同じファイルの末尾にまとめる |

---

## まとめ

- `@Preview`でエミュレータ不要のUI確認
- ライト/ダーク、複数デバイスを同時プレビュー
- `PreviewParameterProvider`で複数データパターン
- カスタムアノテーションでPreview設定を共通化
- インタラクティブモードでクリック・スクロール確認

---

8種類のAndroidアプリテンプレート（Preview設定済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Composeレイアウト入門](https://zenn.dev/myougatheaxo/articles/compose-layout-basics-2026)
- [ダークモード完全対応ガイド](https://zenn.dev/myougatheaxo/articles/android-dark-mode-guide-2026)
- [アクセシビリティ対応入門](https://zenn.dev/myougatheaxo/articles/android-accessibility-basics-2026)
