---
title: "ダークモード完全対応ガイド — Composeでライト/ダーク切り替えを実装する"
emoji: "🌙"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "darkmode", "design"]
published: true
---

## この記事で学べること

ダークモードは2026年のアプリに必須の機能です。Composeなら**5行で自動対応**、手動トグルも簡単に追加できます。

---

## 自動対応（システム設定に追従）

```kotlin
@Composable
fun MyAppTheme(content: @Composable () -> Unit) {
    val darkTheme = isSystemInDarkTheme()

    val colorScheme = if (darkTheme) darkColorScheme() else lightColorScheme()

    MaterialTheme(
        colorScheme = colorScheme,
        content = content
    )
}
```

`isSystemInDarkTheme()`がシステムの設定を読み取り、自動でカラースキームを切り替えます。

---

## 手動トグル（DataStore保存）

```kotlin
class ThemeRepository(private val dataStore: DataStore<Preferences>) {
    private val DARK_MODE_KEY = booleanPreferencesKey("dark_mode")

    val isDarkMode: Flow<Boolean> = dataStore.data
        .map { it[DARK_MODE_KEY] ?: false }

    suspend fun setDarkMode(enabled: Boolean) {
        dataStore.edit { it[DARK_MODE_KEY] = enabled }
    }
}
```

```kotlin
class ThemeViewModel(private val repo: ThemeRepository) : ViewModel() {
    val isDarkMode = repo.isDarkMode
        .stateIn(viewModelScope, SharingStarted.Lazily, false)

    fun toggleDarkMode() {
        viewModelScope.launch {
            repo.setDarkMode(!isDarkMode.value)
        }
    }
}
```

```kotlin
@Composable
fun MyApp(themeViewModel: ThemeViewModel) {
    val isDark by themeViewModel.isDarkMode.collectAsState()

    MyAppTheme(darkTheme = isDark) {
        Scaffold { padding ->
            // メインコンテンツ
        }
    }
}
```

---

## 3つのモード対応

ユーザーに「システム追従 / ライト / ダーク」の3択を提供するパターン：

```kotlin
enum class ThemeMode { SYSTEM, LIGHT, DARK }

@Composable
fun MyAppTheme(
    themeMode: ThemeMode = ThemeMode.SYSTEM,
    content: @Composable () -> Unit
) {
    val darkTheme = when (themeMode) {
        ThemeMode.SYSTEM -> isSystemInDarkTheme()
        ThemeMode.LIGHT -> false
        ThemeMode.DARK -> true
    }

    val colorScheme = if (darkTheme) {
        darkColorScheme(
            primary = Color(0xFFBB86FC),
            secondary = Color(0xFF03DAC5),
            background = Color(0xFF121212),
            surface = Color(0xFF121212)
        )
    } else {
        lightColorScheme(
            primary = Color(0xFF6200EE),
            secondary = Color(0xFF03DAC5),
            background = Color.White,
            surface = Color.White
        )
    }

    MaterialTheme(colorScheme = colorScheme, content = content)
}
```

---

## ダークモードの色設計

### やってはいけないこと

| NG | 理由 |
|----|------|
| 背景を真っ黒(#000000) | OLEDでコントラストが強すぎて目が疲れる |
| 白テキスト(#FFFFFF) | 同上 |
| ライトモードの色をそのまま使う | コントラストが合わない |

### 推奨

| 要素 | ダークモード推奨色 |
|------|------------------|
| 背景 | #121212 |
| Surface | #1E1E1E |
| テキスト | #E0E0E0（87%白） |
| セカンダリテキスト | #A0A0A0（60%白） |

Material3のデフォルトカラーはこれらを考慮済みです。

---

## Dynamic Color（Android 12+）

```kotlin
val colorScheme = when {
    dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
        val context = LocalContext.current
        if (darkTheme) dynamicDarkColorScheme(context)
        else dynamicLightColorScheme(context)
    }
    darkTheme -> DarkColorScheme
    else -> LightColorScheme
}
```

壁紙の色からカラースキームを自動生成。ダークモード時も壁紙に合った暗い色が使われます。

---

## Preview で確認

```kotlin
@Preview(name = "Light", showBackground = true)
@Preview(name = "Dark", showBackground = true, uiMode = Configuration.UI_MODE_NIGHT_YES)
@Composable
fun HomeScreenPreview() {
    MyAppTheme {
        HomeScreen()
    }
}
```

2つの`@Preview`を並べることで、ライト/ダーク両方を同時にプレビュー確認できます。

---

## まとめ

- `isSystemInDarkTheme()`で自動対応（最小5行）
- 手動トグルはDataStore + ViewModelで実装
- 3モード対応（System / Light / Dark）が理想
- 背景は#121212、テキストは#E0E0E0が推奨
- Dynamic Colorでユーザーの壁紙に自動適応

---

8種類のAndroidアプリテンプレート（全てダークモード完全対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Material3テーマ完全ガイド](https://zenn.dev/myougatheaxo/articles/material3-theme-guide-2026)
- [DataStore移行ガイド](https://zenn.dev/myougatheaxo/articles/android-datastore-migration-2026)
- [ViewModel + StateFlow入門](https://zenn.dev/myougatheaxo/articles/android-viewmodel-stateflow-2026)
