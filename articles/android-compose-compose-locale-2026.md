---
title: "Compose Locale完全ガイド — 多言語対応/言語切替/Per-App Language"
emoji: "🌍"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "i18n"]
published: true
---

## この記事で学べること

**Compose Locale**（多言語対応、Per-App Language Preferences、アプリ内言語切替）を解説します。

---

## strings.xml多言語

```xml
<!-- res/values/strings.xml (デフォルト: 日本語) -->
<resources>
    <string name="app_name">マイアプリ</string>
    <string name="greeting">こんにちは、%sさん</string>
    <string name="item_count">%d件のアイテム</string>
</resources>

<!-- res/values-en/strings.xml -->
<resources>
    <string name="app_name">My App</string>
    <string name="greeting">Hello, %s</string>
    <string name="item_count">%d items</string>
</resources>
```

```kotlin
@Composable
fun LocalizedScreen(userName: String) {
    Column(Modifier.padding(16.dp)) {
        Text(stringResource(R.string.greeting, userName))
        Text(stringResource(R.string.item_count, 42))
    }
}
```

---

## Per-App Language（Android 13+）

```kotlin
@Composable
fun LanguageSelector() {
    val context = LocalContext.current
    val currentLocale = AppCompatDelegate.getApplicationLocales()
        .get(0)?.language ?: "ja"

    val languages = listOf(
        "ja" to "日本語",
        "en" to "English",
        "zh" to "中文",
        "ko" to "한국어"
    )

    Column(Modifier.padding(16.dp)) {
        Text("言語設定", style = MaterialTheme.typography.titleLarge)
        Spacer(Modifier.height(16.dp))

        languages.forEach { (code, name) ->
            ListItem(
                headlineContent = { Text(name) },
                leadingContent = {
                    RadioButton(
                        selected = currentLocale == code,
                        onClick = {
                            val localeList = LocaleListCompat.forLanguageTags(code)
                            AppCompatDelegate.setApplicationLocales(localeList)
                        }
                    )
                },
                modifier = Modifier.clickable {
                    val localeList = LocaleListCompat.forLanguageTags(code)
                    AppCompatDelegate.setApplicationLocales(localeList)
                }
            )
        }
    }
}
```

---

## 数値/日付フォーマット

```kotlin
@Composable
fun LocalizedFormat() {
    val locale = LocalConfiguration.current.locales[0]

    val number = 1234567.89
    val date = Date()

    val numberFormat = NumberFormat.getNumberInstance(locale)
    val dateFormat = DateFormat.getDateInstance(DateFormat.LONG, locale)
    val currencyFormat = NumberFormat.getCurrencyInstance(locale)

    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(8.dp)) {
        Text("数値: ${numberFormat.format(number)}")
        Text("日付: ${dateFormat.format(date)}")
        Text("通貨: ${currencyFormat.format(number)}")
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `stringResource` | ローカライズ文字列 |
| `AppCompatDelegate.setApplicationLocales` | 言語切替 |
| `LocalConfiguration` | 現在ロケール取得 |
| `NumberFormat` | 数値フォーマット |

- `values-xx`フォルダで言語別strings.xml
- Per-App Language（API 33+）でシステム設定に統合
- `AppCompatDelegate`で下位互換性のある言語切替
- 数値/日付/通貨はLocale対応フォーマッタを使用

---

8種類のAndroidアプリテンプレート（多言語対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose SettingsScreen](https://zenn.dev/myougatheaxo/articles/android-compose-compose-settings-screen-2026)
- [DataStore](https://zenn.dev/myougatheaxo/articles/android-compose-datastore-2026)
- [Compose DarkTheme](https://zenn.dev/myougatheaxo/articles/android-compose-compose-dark-theme-2026)
