---
title: "Compose Markdown完全ガイド — Markdownレンダリング/コードハイライト/画像表示/カスタムスタイル"
emoji: "📝"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "markdown"]
published: true
---

## この記事で学べること

**Compose Markdown**（Markdownパース、コードハイライト、画像表示、カスタムテーマ）を解説します。

---

## ライブラリ設定

```groovy
dependencies {
    implementation("com.mikepenz:multiplatform-markdown-renderer-m3:0.15.0")
    implementation("com.mikepenz:multiplatform-markdown-renderer-coil3:0.15.0")
}
```

---

## 基本表示

```kotlin
@Composable
fun MarkdownScreen() {
    val markdownContent = """
    # Jetpack Compose

    **Compose**は宣言的UIツールキットです。

    ## 特徴
    - 宣言的構文
    - リアクティブ
    - Kotlin製

    ```kotlin
    @Composable
    fun Hello() {
        Text("Hello Compose!")
    }
    ```

    > Composeは未来のAndroid UIです。

    詳しくは[公式ドキュメント](https://developer.android.com)をご覧ください。
    """.trimIndent()

    Markdown(
        content = markdownContent,
        modifier = Modifier.fillMaxSize().padding(16.dp).verticalScroll(rememberScrollState())
    )
}
```

---

## カスタムスタイル

```kotlin
@Composable
fun StyledMarkdown(content: String) {
    val colors = markdownColor(
        text = MaterialTheme.colorScheme.onSurface,
        codeText = Color(0xFFD4D4D4),
        codeBackground = Color(0xFF1E1E1E),
        linkText = MaterialTheme.colorScheme.primary
    )

    val typography = markdownTypography(
        h1 = MaterialTheme.typography.headlineLarge,
        h2 = MaterialTheme.typography.headlineMedium,
        h3 = MaterialTheme.typography.titleLarge,
        body1 = MaterialTheme.typography.bodyLarge,
        code = MaterialTheme.typography.bodyMedium.copy(fontFamily = FontFamily.Monospace)
    )

    Markdown(
        content = content,
        colors = colors,
        typography = typography,
        modifier = Modifier.fillMaxSize().padding(16.dp)
    )
}
```

---

## チャットメッセージ

```kotlin
@Composable
fun ChatMessage(message: String, isAI: Boolean) {
    Card(
        modifier = Modifier.fillMaxWidth().padding(vertical = 4.dp),
        colors = CardDefaults.cardColors(
            containerColor = if (isAI) MaterialTheme.colorScheme.surfaceVariant
            else MaterialTheme.colorScheme.primaryContainer
        )
    ) {
        Column(Modifier.padding(12.dp)) {
            Text(
                if (isAI) "AI" else "You",
                style = MaterialTheme.typography.labelSmall,
                fontWeight = FontWeight.Bold
            )
            Spacer(Modifier.height(4.dp))
            Markdown(content = message)
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `Markdown` | MD表示 |
| `markdownColor` | 色設定 |
| `markdownTypography` | フォント設定 |
| Coil3連携 | 画像表示 |

- `multiplatform-markdown-renderer`でComposeネイティブのMD表示
- カスタムカラーとタイポグラフィでテーマに合わせた表示
- コードブロック、リンク、画像、テーブルをサポート
- AIチャット等のMarkdownレスポンス表示に最適

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose RichText](https://zenn.dev/myougatheaxo/articles/android-compose-compose-richtext-2026)
- [Compose MarkdownRenderer](https://zenn.dev/myougatheaxo/articles/android-compose-compose-markdown-renderer-2026)
- [Compose TextStyle](https://zenn.dev/myougatheaxo/articles/android-compose-compose-text-style-2026)
