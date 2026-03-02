---
title: "Compose Preview活用ガイド — デバッグ/マルチプレビュー"
emoji: "🔬"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "preview"]
published: true
---

## この記事で学べること

Composeの**Preview機能**（マルチプレビュー、パラメータ、インタラクティブモード）を解説します。

---

## 基本のPreview

```kotlin
@Preview(
    showBackground = true,
    name = "Default Preview"
)
@Composable
fun GreetingPreview() {
    MaterialTheme {
        Greeting("Android")
    }
}

// 背景色指定
@Preview(
    showBackground = true,
    backgroundColor = 0xFF000000,
    name = "Dark Background"
)
@Composable
fun DarkPreview() {
    Greeting("Dark")
}
```

---

## マルチプレビューアノテーション

```kotlin
@Preview(name = "Light", uiMode = Configuration.UI_MODE_NIGHT_NO)
@Preview(name = "Dark", uiMode = Configuration.UI_MODE_NIGHT_YES)
annotation class ThemePreview

@Preview(name = "Phone", widthDp = 360)
@Preview(name = "Tablet", widthDp = 800)
@Preview(name = "Foldable", widthDp = 600)
annotation class DevicePreview

@Preview(name = "Small Font", fontScale = 0.85f)
@Preview(name = "Normal Font", fontScale = 1f)
@Preview(name = "Large Font", fontScale = 1.5f)
annotation class FontScalePreview

// 使用: 1つのアノテーションで複数プレビュー
@ThemePreview
@DevicePreview
@Composable
fun MyScreenPreview() {
    MyAppTheme {
        MyScreen()
    }
}
```

---

## PreviewParameter

```kotlin
class SampleUserProvider : PreviewParameterProvider<User> {
    override val values = sequenceOf(
        User("Alice", "alice@example.com", true),
        User("Bob", "bob@example.com", false),
        User("", "", false) // 空の状態
    )
}

@Preview(showBackground = true)
@Composable
fun UserCardPreview(
    @PreviewParameter(SampleUserProvider::class) user: User
) {
    MaterialTheme {
        UserCard(user)
    }
}
```

---

## Preview用のデータ提供

```kotlin
// Previewで使うサンプルデータ
object PreviewData {
    val sampleArticle = Article(
        id = "1",
        title = "サンプル記事タイトル",
        content = "記事の本文がここに入ります。",
        author = "Author",
        createdAt = System.currentTimeMillis()
    )

    val sampleArticles = (1..10).map { i ->
        sampleArticle.copy(id = "$i", title = "記事 $i")
    }
}

@Preview
@Composable
fun ArticleListPreview() {
    MaterialTheme {
        ArticleList(
            articles = PreviewData.sampleArticles,
            onArticleClick = {}
        )
    }
}
```

---

## Previewでのインタラクション

```kotlin
// @Preview(showSystemUi = true) でシステムUI表示
@Preview(
    showSystemUi = true,
    device = Devices.PIXEL_7
)
@Composable
fun FullScreenPreview() {
    MyAppTheme {
        Scaffold(
            topBar = { TopAppBar(title = { Text("Preview") }) },
            bottomBar = { NavigationBar { /* items */ } }
        ) { padding ->
            Box(Modifier.padding(padding)) {
                Text("Full screen preview")
            }
        }
    }
}
```

---

## まとめ

- `@Preview`で即座にUI確認（ビルド不要）
- カスタムアノテーションで複数テーマ/デバイスを一括プレビュー
- `PreviewParameterProvider`でデータバリエーション
- `showSystemUi = true`でステータスバー/ナビバー表示
- `fontScale`でフォントサイズのアクセシビリティ確認
- Preview用サンプルデータはobjectにまとめて管理

---

8種類のAndroidアプリテンプレート（Preview設定済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [スクリーンショットテスト](https://zenn.dev/myougatheaxo/articles/android-compose-testing-screenshot-2026)
- [UIテスト](https://zenn.dev/myougatheaxo/articles/android-compose-testing-ui-2026)
- [マルチスクリーンサイズ](https://zenn.dev/myougatheaxo/articles/android-compose-multi-screen-size-2026)
