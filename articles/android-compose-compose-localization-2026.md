---
title: "Compose Localization完全ガイド — 多言語対応/stringResource/複数形/RTL/動的言語切替"
emoji: "🌍"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "localization"]
published: true
---

## この記事で学べること

**Compose Localization**（多言語対応、stringResource、複数形、RTLレイアウト、動的言語切替）を解説します。

---

## stringResource

```kotlin
@Composable
fun LocalizedScreen() {
    Column(Modifier.fillMaxSize().padding(16.dp)) {
        Text(stringResource(R.string.app_name), style = MaterialTheme.typography.headlineMedium)
        Spacer(Modifier.height(16.dp))
        Text(stringResource(R.string.welcome_message, "みょうが"))
        Spacer(Modifier.height(8.dp))
        Text(pluralStringResource(R.plurals.items_count, 5, 5))
    }
}
```

```xml
<!-- res/values/strings.xml (日本語) -->
<resources>
    <string name="app_name">マイアプリ</string>
    <string name="welcome_message">%1$sさん、ようこそ！</string>
    <plurals name="items_count">
        <item quantity="other">%d件のアイテム</item>
    </plurals>
</resources>

<!-- res/values-en/strings.xml (英語) -->
<resources>
    <string name="app_name">My App</string>
    <string name="welcome_message">Welcome, %1$s!</string>
    <plurals name="items_count">
        <item quantity="one">%d item</item>
        <item quantity="other">%d items</item>
    </plurals>
</resources>
```

---

## 動的言語切替

```kotlin
@Composable
fun LanguageSelector(onLanguageChange: (String) -> Unit) {
    val languages = listOf("ja" to "日本語", "en" to "English", "zh" to "中文")
    var expanded by remember { mutableStateOf(false) }
    val currentLocale = LocalContext.current.resources.configuration.locales[0]

    ExposedDropdownMenuBox(expanded = expanded, onExpandedChange = { expanded = it }) {
        OutlinedTextField(
            value = languages.find { it.first == currentLocale.language }?.second ?: "日本語",
            onValueChange = {},
            readOnly = true,
            label = { Text("言語") },
            trailingIcon = { ExposedDropdownMenuDefaults.TrailingIcon(expanded) },
            modifier = Modifier.menuAnchor().fillMaxWidth()
        )
        ExposedDropdownMenu(expanded = expanded, onDismissRequest = { expanded = false }) {
            languages.forEach { (code, name) ->
                DropdownMenuItem(
                    text = { Text(name) },
                    onClick = { onLanguageChange(code); expanded = false }
                )
            }
        }
    }
}

// Activity
fun changeLanguage(context: Context, languageCode: String) {
    val locale = Locale(languageCode)
    val config = context.resources.configuration.apply {
        setLocale(locale)
    }
    context.createConfigurationContext(config)
    AppCompatDelegate.setApplicationLocales(LocaleListCompat.forLanguageTags(languageCode))
}
```

---

## 日付/数値フォーマット

```kotlin
@Composable
fun FormattedContent() {
    val locale = LocalContext.current.resources.configuration.locales[0]
    val now = remember { Date() }

    Column(Modifier.padding(16.dp)) {
        Text("日付: ${DateFormat.getDateInstance(DateFormat.LONG, locale).format(now)}")
        Text("通貨: ${NumberFormat.getCurrencyInstance(locale).format(1234.56)}")
        Text("数値: ${NumberFormat.getNumberInstance(locale).format(1234567.89)}")
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `stringResource` | 文字列リソース |
| `pluralStringResource` | 複数形 |
| `AppCompatDelegate` | 言語切替 |
| `DateFormat` | 日付フォーマット |

- `stringResource`でロケールに応じた文字列を取得
- `pluralStringResource`で複数形を正しく処理
- `AppCompatDelegate.setApplicationLocales`で動的言語切替
- 日付/通貨/数値はロケール対応フォーマッターを使用

---

8種類のAndroidアプリテンプレート（多言語対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose RTL](https://zenn.dev/myougatheaxo/articles/android-compose-compose-rtl-2026)
- [Compose Accessibility](https://zenn.dev/myougatheaxo/articles/android-compose-compose-accessibility-2026)
- [Compose Typography](https://zenn.dev/myougatheaxo/articles/android-compose-compose-typography-2026)
