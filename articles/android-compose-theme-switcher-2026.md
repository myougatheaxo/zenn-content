---
title: "テーマ切替実装ガイド — ライト/ダーク/Dynamic Color切替"
emoji: "🌓"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "theme"]
published: true
---

## この記事で学べること

Composeでの**テーマ切替**（ライト/ダーク/システム設定/Dynamic Color）の実装を解説します。

---

## テーマ設定の保存

```kotlin
enum class ThemeMode { LIGHT, DARK, SYSTEM }

class ThemeRepository(private val dataStore: DataStore<Preferences>) {
    private val THEME_KEY = stringPreferencesKey("theme_mode")
    private val DYNAMIC_COLOR_KEY = booleanPreferencesKey("dynamic_color")

    val themeMode: Flow<ThemeMode> = dataStore.data.map { prefs ->
        ThemeMode.valueOf(prefs[THEME_KEY] ?: ThemeMode.SYSTEM.name)
    }

    val dynamicColor: Flow<Boolean> = dataStore.data.map { prefs ->
        prefs[DYNAMIC_COLOR_KEY] ?: true
    }

    suspend fun setThemeMode(mode: ThemeMode) {
        dataStore.edit { it[THEME_KEY] = mode.name }
    }

    suspend fun setDynamicColor(enabled: Boolean) {
        dataStore.edit { it[DYNAMIC_COLOR_KEY] = enabled }
    }
}
```

---

## テーマComposable

```kotlin
@Composable
fun AppTheme(
    themeMode: ThemeMode = ThemeMode.SYSTEM,
    dynamicColor: Boolean = true,
    content: @Composable () -> Unit
) {
    val darkTheme = when (themeMode) {
        ThemeMode.LIGHT -> false
        ThemeMode.DARK -> true
        ThemeMode.SYSTEM -> isSystemInDarkTheme()
    }

    val colorScheme = when {
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            val context = LocalContext.current
            if (darkTheme) dynamicDarkColorScheme(context)
            else dynamicLightColorScheme(context)
        }
        darkTheme -> darkColorScheme()
        else -> lightColorScheme()
    }

    MaterialTheme(
        colorScheme = colorScheme,
        typography = AppTypography,
        content = content
    )
}
```

---

## Application/Activity統合

```kotlin
@Composable
fun AppRoot(themeViewModel: ThemeViewModel = viewModel()) {
    val themeMode by themeViewModel.themeMode.collectAsStateWithLifecycle()
    val dynamicColor by themeViewModel.dynamicColor.collectAsStateWithLifecycle()

    AppTheme(
        themeMode = themeMode,
        dynamicColor = dynamicColor
    ) {
        AppNavHost()
    }
}
```

---

## テーマ設定画面

```kotlin
@Composable
fun ThemeSettingsScreen(viewModel: ThemeViewModel = viewModel()) {
    val themeMode by viewModel.themeMode.collectAsStateWithLifecycle()
    val dynamicColor by viewModel.dynamicColor.collectAsStateWithLifecycle()

    Column(Modifier.padding(16.dp)) {
        Text("テーマ設定", style = MaterialTheme.typography.titleMedium)
        Spacer(Modifier.height(16.dp))

        // テーマモード選択
        ThemeMode.entries.forEach { mode ->
            Row(
                Modifier
                    .fillMaxWidth()
                    .clickable { viewModel.setThemeMode(mode) }
                    .padding(vertical = 12.dp),
                verticalAlignment = Alignment.CenterVertically
            ) {
                RadioButton(
                    selected = themeMode == mode,
                    onClick = { viewModel.setThemeMode(mode) }
                )
                Spacer(Modifier.width(8.dp))
                Text(
                    when (mode) {
                        ThemeMode.LIGHT -> "ライトモード"
                        ThemeMode.DARK -> "ダークモード"
                        ThemeMode.SYSTEM -> "システム設定に従う"
                    }
                )
            }
        }

        HorizontalDivider(Modifier.padding(vertical = 8.dp))

        // Dynamic Color
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
            ListItem(
                headlineContent = { Text("ダイナミックカラー") },
                supportingContent = { Text("壁紙に合わせたカラーを使用") },
                trailingContent = {
                    Switch(
                        checked = dynamicColor,
                        onCheckedChange = { viewModel.setDynamicColor(it) }
                    )
                },
                modifier = Modifier.clickable { viewModel.setDynamicColor(!dynamicColor) }
            )
        }

        // プレビュー
        Spacer(Modifier.height(16.dp))
        Text("プレビュー", style = MaterialTheme.typography.titleSmall)
        Card(Modifier.fillMaxWidth().padding(vertical = 8.dp)) {
            Column(Modifier.padding(16.dp)) {
                Text("サンプルテキスト", style = MaterialTheme.typography.titleMedium)
                Text("これはプレビュー表示です", style = MaterialTheme.typography.bodyMedium)
                Spacer(Modifier.height(8.dp))
                Button(onClick = {}) { Text("ボタン") }
            }
        }
    }
}
```

---

## まとめ

- `DataStore`でテーマ設定を永続化
- `ThemeMode`(LIGHT/DARK/SYSTEM)で3モード対応
- `dynamicColorScheme()`でMaterial You対応
- `isSystemInDarkTheme()`でシステム設定を取得
- 設定画面でRadioButton/Switch切り替え
- プレビューで変更を即座に確認

---

8種類のAndroidアプリテンプレート（テーマ切替設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ダークモード対応ガイド](https://zenn.dev/myougatheaxo/articles/android-dark-mode-guide-2026)
- [Dynamic Color/Material Youガイド](https://zenn.dev/myougatheaxo/articles/android-compose-theming-dynamic-2026)
- [設定画面実装ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-settings-screen-2026)
