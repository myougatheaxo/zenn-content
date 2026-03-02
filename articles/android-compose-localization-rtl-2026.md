---
title: "多言語/RTL対応ガイド — Compose多言語化とRTLレイアウト"
emoji: "🌍"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "i18n"]
published: true
---

## この記事で学べること

Composeでの**多言語対応**とRTL（右から左）レイアウト対応を解説します。

---

## 文字列リソース

```xml
<!-- res/values/strings.xml (デフォルト: 日本語) -->
<resources>
    <string name="app_name">マイアプリ</string>
    <string name="greeting">こんにちは、%1$s さん！</string>
    <string name="item_count">%1$d 件</string>
</resources>

<!-- res/values-en/strings.xml (英語) -->
<resources>
    <string name="app_name">My App</string>
    <string name="greeting">Hello, %1$s!</string>
    <string name="item_count">%1$d items</string>
</resources>
```

---

## Compose での使用

```kotlin
@Composable
fun LocalizedContent() {
    Column(Modifier.padding(16.dp)) {
        Text(stringResource(R.string.greeting, "Alice"))

        val count = 5
        Text(stringResource(R.string.item_count, count))

        // 複数形
        Text(pluralStringResource(R.plurals.item_count_plural, count, count))
    }
}
```

---

## アプリ内言語切替

```kotlin
// Android 13+ Per-App Language
fun changeLanguage(context: Context, languageTag: String) {
    val appLocale = LocaleListCompat.forLanguageTags(languageTag)
    AppCompatDelegate.setApplicationLocales(appLocale)
}

@Composable
fun LanguageSwitcher() {
    val languages = listOf(
        "ja" to "日本語",
        "en" to "English",
        "zh" to "中文",
        "ko" to "한국어"
    )
    val context = LocalContext.current

    Column {
        languages.forEach { (tag, name) ->
            ListItem(
                headlineContent = { Text(name) },
                modifier = Modifier.clickable { changeLanguage(context, tag) }
            )
        }
    }
}
```

---

## RTL対応

```kotlin
@Composable
fun RtlAwareLayout() {
    // 自動でRTL対応されるModifier
    Row(Modifier.padding(start = 16.dp, end = 8.dp)) {
        // start/endはRTLで自動反転
        Icon(Icons.AutoMirrored.Default.ArrowBack, "戻る") // 自動ミラー
        Text("テキスト")
    }
}

// RTLプレビュー
@Preview(locale = "ar") // アラビア語RTL
@Composable
fun PreviewRtl() {
    AppTheme {
        RtlAwareLayout()
    }
}
```

---

## まとめ

- `res/values-{lang}/strings.xml`で言語別リソース
- `stringResource()`でCompose内から参照
- `pluralStringResource()`で複数形対応
- `AppCompatDelegate.setApplicationLocales()`で動的切替
- `start`/`end`でRTL自動対応
- `Icons.AutoMirrored`でアイコンの自動反転

---

8種類のAndroidアプリテンプレート（多言語対応設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [多言語対応ガイド](https://zenn.dev/myougatheaxo/articles/android-localization-i18n-2026)
- [文字列操作ガイド](https://zenn.dev/myougatheaxo/articles/kotlin-string-template-2026)
- [設定画面実装ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-settings-screen-2026)
