---
title: "動的テーマ切替完全ガイド — ユーザー設定/ダークモード/カラーシード"
emoji: "🎭"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "design"]
published: true
---

## この記事で学べること

**動的テーマ切替**（ユーザー設定による切替、DataStore永続化、カラーシード、Material3テーマ生成）を解説します。

---

## テーマ設定管理

```kotlin
enum class ThemeMode { SYSTEM, LIGHT, DARK }

class ThemeRepository @Inject constructor(
    private val dataStore: DataStore<Preferences>
) {
    val themeMode: Flow<ThemeMode> = dataStore.data
        .map { prefs ->
            val value = prefs[stringPreferencesKey("theme_mode")] ?: "SYSTEM"
            ThemeMode.valueOf(value)
        }

    val seedColor: Flow<Color> = dataStore.data
        .map { prefs ->
            val colorInt = prefs[intPreferencesKey("seed_color")] ?: 0xFF6200EE.toInt()
            Color(colorInt)
        }

    suspend fun setThemeMode(mode: ThemeMode) {
        dataStore.edit { it[stringPreferencesKey("theme_mode")] = mode.name }
    }

    suspend fun setSeedColor(color: Color) {
        dataStore.edit { it[intPreferencesKey("seed_color")] = color.toArgb() }
    }
}
```

---

## テーマ適用

```kotlin
@Composable
fun DynamicThemeApp(
    themeRepository: ThemeRepository,
    content: @Composable () -> Unit
) {
    val themeMode by themeRepository.themeMode.collectAsStateWithLifecycle(ThemeMode.SYSTEM)
    val seedColor by themeRepository.seedColor.collectAsStateWithLifecycle(Color(0xFF6200EE))

    val isDarkTheme = when (themeMode) {
        ThemeMode.SYSTEM -> isSystemInDarkTheme()
        ThemeMode.LIGHT -> false
        ThemeMode.DARK -> true
    }

    val colorScheme = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
        val context = LocalContext.current
        if (isDarkTheme) dynamicDarkColorScheme(context)
        else dynamicLightColorScheme(context)
    } else {
        generateColorScheme(seedColor, isDarkTheme)
    }

    MaterialTheme(colorScheme = colorScheme) { content() }
}

fun generateColorScheme(seed: Color, isDark: Boolean): ColorScheme {
    return if (isDark) {
        darkColorScheme(
            primary = seed,
            secondary = seed.copy(alpha = 0.7f),
            background = Color(0xFF121212)
        )
    } else {
        lightColorScheme(
            primary = seed,
            secondary = seed.copy(alpha = 0.7f),
            background = Color(0xFFFFFBFE)
        )
    }
}
```

---

## テーマ設定画面

```kotlin
@Composable
fun ThemeSettingsScreen(viewModel: ThemeViewModel = hiltViewModel()) {
    val currentMode by viewModel.themeMode.collectAsStateWithLifecycle(ThemeMode.SYSTEM)
    val currentColor by viewModel.seedColor.collectAsStateWithLifecycle(Color(0xFF6200EE))

    Column(Modifier.fillMaxSize().padding(16.dp)) {
        Text("テーマ設定", style = MaterialTheme.typography.headlineSmall)
        Spacer(Modifier.height(16.dp))

        // モード選択
        ThemeMode.entries.forEach { mode ->
            Row(
                Modifier.fillMaxWidth().clickable { viewModel.setThemeMode(mode) }.padding(12.dp),
                verticalAlignment = Alignment.CenterVertically
            ) {
                RadioButton(selected = currentMode == mode, onClick = { viewModel.setThemeMode(mode) })
                Spacer(Modifier.width(8.dp))
                Text(when (mode) {
                    ThemeMode.SYSTEM -> "システム設定に従う"
                    ThemeMode.LIGHT -> "ライト"
                    ThemeMode.DARK -> "ダーク"
                })
            }
        }

        Spacer(Modifier.height(24.dp))
        Text("テーマカラー", style = MaterialTheme.typography.titleMedium)
        Spacer(Modifier.height(8.dp))

        // カラーパレット
        val colors = listOf(
            Color(0xFF6200EE), Color(0xFF03DAC5), Color(0xFFFF5722),
            Color(0xFF4CAF50), Color(0xFF2196F3), Color(0xFFE91E63)
        )
        Row(horizontalArrangement = Arrangement.spacedBy(12.dp)) {
            colors.forEach { color ->
                Box(
                    Modifier
                        .size(48.dp)
                        .clip(CircleShape)
                        .background(color)
                        .border(
                            if (currentColor == color) 3.dp else 0.dp,
                            MaterialTheme.colorScheme.onSurface, CircleShape
                        )
                        .clickable { viewModel.setSeedColor(color) }
                )
            }
        }
    }
}
```

---

## まとめ

| 機能 | 実装 |
|------|------|
| モード切替 | `ThemeMode` enum + DataStore |
| Dynamic Color | `dynamicColorScheme` (API 31+) |
| カスタムカラー | `generateColorScheme` |
| 永続化 | DataStore Preferences |

- DataStoreでテーマ設定を永続化
- `isSystemInDarkTheme()`でシステム設定に追従
- API 31+はDynamic Color、それ以外はカスタムカラー
- カラーピッカーでユーザー好みのテーマに

---

8種類のAndroidアプリテンプレート（テーマカスタマイズ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Material3テーマ](https://zenn.dev/myougatheaxo/articles/android-compose-material3-theme-2026)
- [カラーパレット](https://zenn.dev/myougatheaxo/articles/android-compose-color-palette-2026)
- [DataStore](https://zenn.dev/myougatheaxo/articles/android-compose-datastore-preferences-2026)
