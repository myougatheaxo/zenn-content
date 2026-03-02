---
title: "多言語対応(i18n)完全ガイド — stringResource/Locale/RTL"
emoji: "🌏"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "i18n"]
published: true
---

## この記事で学べること

**多言語対応**（stringResource、Locale切替、複数形、RTLレイアウト、テスト）を解説します。

---

## strings.xml

```xml
<!-- res/values/strings.xml (デフォルト: 英語) -->
<resources>
    <string name="app_name">My App</string>
    <string name="greeting">Hello, %1$s!</string>
    <string name="item_count">%1$d items</string>
</resources>

<!-- res/values-ja/strings.xml (日本語) -->
<resources>
    <string name="app_name">マイアプリ</string>
    <string name="greeting">こんにちは、%1$sさん！</string>
    <string name="item_count">%1$d 個のアイテム</string>
</resources>
```

---

## Composeでの使用

```kotlin
@Composable
fun GreetingScreen(userName: String) {
    Column(Modifier.padding(16.dp)) {
        Text(
            text = stringResource(R.string.greeting, userName),
            style = MaterialTheme.typography.headlineMedium
        )

        Text(
            text = pluralStringResource(
                R.plurals.notification_count,
                count = 5,
                5
            )
        )
    }
}
```

---

## アプリ内Locale切替

```kotlin
class LanguageRepository @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val Context.dataStore by preferencesDataStore("settings")
    private val LANG_KEY = stringPreferencesKey("language")

    val language: Flow<String> = context.dataStore.data.map { prefs ->
        prefs[LANG_KEY] ?: "system"
    }

    suspend fun setLanguage(langCode: String) {
        context.dataStore.edit { it[LANG_KEY] = langCode }
    }
}

// AppCompatDelegate で言語変更
fun applyLanguage(context: Context, langCode: String) {
    val locale = if (langCode == "system") {
        AppCompatDelegate.setApplicationLocales(LocaleListCompat.getEmptyLocaleList())
        return
    } else {
        AppCompatDelegate.setApplicationLocales(
            LocaleListCompat.forLanguageTags(langCode)
        )
    }
}
```

---

## 言語選択画面

```kotlin
@Composable
fun LanguageSettingsScreen(viewModel: SettingsViewModel = hiltViewModel()) {
    val currentLang by viewModel.language.collectAsStateWithLifecycle()

    val languages = listOf(
        "system" to "システム設定",
        "ja" to "日本語",
        "en" to "English",
        "ko" to "한국어",
        "zh" to "中文"
    )

    Column(Modifier.padding(16.dp)) {
        Text("言語設定", style = MaterialTheme.typography.headlineMedium)
        Spacer(Modifier.height(16.dp))

        languages.forEach { (code, label) ->
            Row(
                Modifier.fillMaxWidth().clickable { viewModel.setLanguage(code) }.padding(12.dp),
                verticalAlignment = Alignment.CenterVertically
            ) {
                RadioButton(selected = currentLang == code, onClick = { viewModel.setLanguage(code) })
                Spacer(Modifier.width(12.dp))
                Text(label)
            }
        }
    }
}
```

---

## 複数形(plurals)

```xml
<!-- res/values/strings.xml -->
<plurals name="notification_count">
    <item quantity="zero">No notifications</item>
    <item quantity="one">%1$d notification</item>
    <item quantity="other">%1$d notifications</item>
</plurals>

<!-- res/values-ja/strings.xml -->
<plurals name="notification_count">
    <item quantity="other">%1$d件の通知</item>
</plurals>
```

---

## まとめ

| 機能 | 実装 |
|------|------|
| 文字列取得 | `stringResource()` |
| フォーマット | `stringResource(R.string.x, arg)` |
| 複数形 | `pluralStringResource()` |
| Locale切替 | `AppCompatDelegate.setApplicationLocales` |
| RTL対応 | `Modifier.padding(start=, end=)` |

- `values-ja/`で言語別リソース
- `%1$s`でフォーマット引数
- `AppCompatDelegate`でアプリ内言語切替
- `start/end`で`left/right`の代わりにRTL対応

---

8種類のAndroidアプリテンプレート（多言語対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [アクセシビリティ](https://zenn.dev/myougatheaxo/articles/android-compose-accessibility-basics-2026)
- [DataStore](https://zenn.dev/myougatheaxo/articles/android-compose-datastore-preferences-2026)
- [カスタムLint](https://zenn.dev/myougatheaxo/articles/android-compose-custom-lint-rule-2026)
