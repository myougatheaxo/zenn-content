---
title: "Compose DarkTheme完全ガイド — ダークモード/テーマ切替/永続化"
emoji: "🌙"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**Compose DarkTheme**（ダークモード対応、テーマ切替、DataStore永続化、Surface Tonal Elevation）を解説します。

---

## 基本ダークテーマ

```kotlin
@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    content: @Composable () -> Unit
) {
    val colorScheme = if (darkTheme) {
        darkColorScheme(
            primary = Color(0xFFBB86FC),
            secondary = Color(0xFF03DAC5),
            background = Color(0xFF121212),
            surface = Color(0xFF1E1E1E)
        )
    } else {
        lightColorScheme(
            primary = Color(0xFF6200EE),
            secondary = Color(0xFF03DAC5),
            background = Color(0xFFFAFAFA),
            surface = Color.White
        )
    }

    MaterialTheme(colorScheme = colorScheme, content = content)
}
```

---

## テーマ設定の永続化

```kotlin
class ThemePreferences @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val dataStore = context.dataStore

    val themeMode: Flow<ThemeMode> = dataStore.data.map { prefs ->
        ThemeMode.valueOf(prefs[stringPreferencesKey("theme")] ?: ThemeMode.SYSTEM.name)
    }

    suspend fun setThemeMode(mode: ThemeMode) {
        dataStore.edit { it[stringPreferencesKey("theme")] = mode.name }
    }
}

enum class ThemeMode { SYSTEM, LIGHT, DARK }

@Composable
fun AppWithTheme(themePrefs: ThemePreferences) {
    val themeMode by themePrefs.themeMode.collectAsStateWithLifecycle(ThemeMode.SYSTEM)

    val darkTheme = when (themeMode) {
        ThemeMode.SYSTEM -> isSystemInDarkTheme()
        ThemeMode.LIGHT -> false
        ThemeMode.DARK -> true
    }

    AppTheme(darkTheme = darkTheme) {
        MainScreen()
    }
}
```

---

## テーマ切替UI

```kotlin
@Composable
fun ThemeSettingsScreen(viewModel: SettingsViewModel = hiltViewModel()) {
    val themeMode by viewModel.themeMode.collectAsStateWithLifecycle()

    Column(Modifier.padding(16.dp)) {
        Text("外観", style = MaterialTheme.typography.titleLarge)
        Spacer(Modifier.height(16.dp))

        listOf(
            ThemeMode.SYSTEM to "システム設定に従う",
            ThemeMode.LIGHT to "ライトモード",
            ThemeMode.DARK to "ダークモード"
        ).forEach { (mode, label) ->
            ListItem(
                headlineContent = { Text(label) },
                leadingContent = {
                    RadioButton(selected = themeMode == mode, onClick = { viewModel.setTheme(mode) })
                },
                modifier = Modifier.clickable { viewModel.setTheme(mode) }
            )
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `isSystemInDarkTheme()` | システム設定取得 |
| `darkColorScheme()` | ダークカラー定義 |
| `lightColorScheme()` | ライトカラー定義 |
| DataStore | テーマ設定永続化 |

- `isSystemInDarkTheme()`でシステム設定を取得
- DataStoreでユーザーのテーマ選択を永続化
- 3モード（システム/ライト/ダーク）切替が標準
- Dynamic Colorとの組み合わせも可能

---

8種類のAndroidアプリテンプレート（ダークモード対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose MaterialYou](https://zenn.dev/myougatheaxo/articles/android-compose-compose-material-you-2026)
- [Compose ColorScheme](https://zenn.dev/myougatheaxo/articles/android-compose-compose-color-scheme-2026)
- [DataStore](https://zenn.dev/myougatheaxo/articles/android-compose-datastore-2026)
