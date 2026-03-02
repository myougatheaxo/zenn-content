---
title: "ダークモード完全ガイド — テーマ切替/永続化/Dynamic Color"
emoji: "🌙"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "darkmode"]
published: true
---

## この記事で学べること

**ダークモード**（テーマ切替、DataStore永続化、Dynamic Color、カスタムカラー）を解説します。

---

## テーマ切替

```kotlin
enum class ThemeMode { SYSTEM, LIGHT, DARK }

@Composable
fun AppTheme(
    themeMode: ThemeMode = ThemeMode.SYSTEM,
    content: @Composable () -> Unit
) {
    val isDark = when (themeMode) {
        ThemeMode.SYSTEM -> isSystemInDarkTheme()
        ThemeMode.LIGHT -> false
        ThemeMode.DARK -> true
    }

    val colorScheme = when {
        Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            val context = LocalContext.current
            if (isDark) dynamicDarkColorScheme(context)
            else dynamicLightColorScheme(context)
        }
        isDark -> darkColorScheme(
            primary = Color(0xFFBB86FC),
            secondary = Color(0xFF03DAC5),
            background = Color(0xFF121212)
        )
        else -> lightColorScheme(
            primary = Color(0xFF6200EE),
            secondary = Color(0xFF03DAC6),
            background = Color(0xFFFFFBFE)
        )
    }

    MaterialTheme(
        colorScheme = colorScheme,
        typography = Typography(),
        content = content
    )
}
```

---

## DataStore永続化

```kotlin
class ThemeRepository @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val Context.dataStore by preferencesDataStore("settings")
    private val THEME_KEY = stringPreferencesKey("theme_mode")

    val themeMode: Flow<ThemeMode> = context.dataStore.data.map { prefs ->
        when (prefs[THEME_KEY]) {
            "light" -> ThemeMode.LIGHT
            "dark" -> ThemeMode.DARK
            else -> ThemeMode.SYSTEM
        }
    }

    suspend fun setThemeMode(mode: ThemeMode) {
        context.dataStore.edit { prefs ->
            prefs[THEME_KEY] = when (mode) {
                ThemeMode.LIGHT -> "light"
                ThemeMode.DARK -> "dark"
                ThemeMode.SYSTEM -> "system"
            }
        }
    }
}
```

---

## 設定画面

```kotlin
@Composable
fun ThemeSettingsScreen(viewModel: SettingsViewModel = hiltViewModel()) {
    val currentTheme by viewModel.themeMode.collectAsStateWithLifecycle()

    Column(Modifier.padding(16.dp)) {
        Text("テーマ設定", style = MaterialTheme.typography.headlineMedium)
        Spacer(Modifier.height(16.dp))

        ThemeMode.entries.forEach { mode ->
            Row(
                Modifier
                    .fillMaxWidth()
                    .clickable { viewModel.setTheme(mode) }
                    .padding(vertical = 12.dp),
                verticalAlignment = Alignment.CenterVertically
            ) {
                RadioButton(
                    selected = currentTheme == mode,
                    onClick = { viewModel.setTheme(mode) }
                )
                Spacer(Modifier.width(12.dp))
                Text(
                    when (mode) {
                        ThemeMode.SYSTEM -> "システム設定に従う"
                        ThemeMode.LIGHT -> "ライト"
                        ThemeMode.DARK -> "ダーク"
                    }
                )
            }
        }
    }
}
```

---

## Activity統合

```kotlin
class MainActivity : ComponentActivity() {
    private val settingsViewModel: SettingsViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent {
            val themeMode by settingsViewModel.themeMode
                .collectAsStateWithLifecycle(ThemeMode.SYSTEM)

            AppTheme(themeMode = themeMode) {
                AppNavigation()
            }
        }
    }
}
```

---

## まとめ

| 機能 | 実装 |
|------|------|
| システム連動 | `isSystemInDarkTheme()` |
| Dynamic Color | `dynamicDarkColorScheme()` |
| 永続化 | `DataStore` |
| テーマ切替 | `ThemeMode` enum |

- `isSystemInDarkTheme()`でシステム設定追従
- Dynamic Color（Android 12+）で壁紙連動
- DataStoreでユーザー設定を永続化
- `ThemeMode.SYSTEM/LIGHT/DARK`の3択

---

8種類のAndroidアプリテンプレート（ダークモード対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Material3テーマ](https://zenn.dev/myougatheaxo/articles/android-compose-material3-theming-2026)
- [DataStore](https://zenn.dev/myougatheaxo/articles/android-compose-datastore-preferences-2026)
- [Edge-to-Edge](https://zenn.dev/myougatheaxo/articles/android-compose-edge-to-edge-2026)
