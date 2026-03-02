---
title: "Compose Material You完全ガイド — Dynamic Color/壁紙テーマ/M3カスタマイズ"
emoji: "🎨"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**Compose Material You**（Dynamic Color、壁紙ベーステーマ、M3カラースキーム、カスタムテーマ設計）を解説します。

---

## Dynamic Color

```kotlin
@Composable
fun MyApp() {
    val dynamicColor = Build.VERSION.SDK_INT >= Build.VERSION_CODES.S
    val darkTheme = isSystemInDarkTheme()

    val colorScheme = when {
        dynamicColor && darkTheme -> dynamicDarkColorScheme(LocalContext.current)
        dynamicColor && !darkTheme -> dynamicLightColorScheme(LocalContext.current)
        darkTheme -> darkColorScheme(
            primary = Color(0xFFBB86FC),
            secondary = Color(0xFF03DAC5)
        )
        else -> lightColorScheme(
            primary = Color(0xFF6200EE),
            secondary = Color(0xFF03DAC5)
        )
    }

    MaterialTheme(colorScheme = colorScheme) {
        Surface(color = MaterialTheme.colorScheme.background) {
            MainScreen()
        }
    }
}
```

---

## カスタムカラースキーム

```kotlin
private val LightColors = lightColorScheme(
    primary = Color(0xFF4CAF50),
    onPrimary = Color.White,
    primaryContainer = Color(0xFFC8E6C9),
    onPrimaryContainer = Color(0xFF1B5E20),
    secondary = Color(0xFF2196F3),
    onSecondary = Color.White,
    background = Color(0xFFFAFAFA),
    surface = Color.White,
    error = Color(0xFFB00020)
)

private val DarkColors = darkColorScheme(
    primary = Color(0xFF81C784),
    onPrimary = Color(0xFF003300),
    primaryContainer = Color(0xFF2E7D32),
    onPrimaryContainer = Color(0xFFC8E6C9),
    secondary = Color(0xFF64B5F6),
    onSecondary = Color(0xFF003366),
    background = Color(0xFF121212),
    surface = Color(0xFF1E1E1E),
    error = Color(0xFFCF6679)
)

@Composable
fun AppTheme(darkTheme: Boolean = isSystemInDarkTheme(), content: @Composable () -> Unit) {
    MaterialTheme(
        colorScheme = if (darkTheme) DarkColors else LightColors,
        typography = Typography(),
        content = content
    )
}
```

---

## テーマカラー使用例

```kotlin
@Composable
fun ThemedScreen() {
    Column(Modifier.fillMaxSize().background(MaterialTheme.colorScheme.background).padding(16.dp)) {
        Card(colors = CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.primaryContainer)) {
            Text(
                "Primary Container",
                Modifier.padding(16.dp),
                color = MaterialTheme.colorScheme.onPrimaryContainer
            )
        }
        Spacer(Modifier.height(8.dp))
        Card(colors = CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.secondaryContainer)) {
            Text(
                "Secondary Container",
                Modifier.padding(16.dp),
                color = MaterialTheme.colorScheme.onSecondaryContainer
            )
        }
        Spacer(Modifier.height(8.dp))
        Card(colors = CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.tertiaryContainer)) {
            Text(
                "Tertiary Container",
                Modifier.padding(16.dp),
                color = MaterialTheme.colorScheme.onTertiaryContainer
            )
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `dynamicLightColorScheme` | 壁紙ライトテーマ |
| `dynamicDarkColorScheme` | 壁紙ダークテーマ |
| `lightColorScheme` | カスタムライト |
| `darkColorScheme` | カスタムダーク |

- Dynamic ColorはAndroid 12+で利用可能
- フォールバック用にカスタムカラースキームを用意
- `MaterialTheme.colorScheme`でテーマカラーにアクセス
- `primaryContainer`/`onPrimaryContainer`ペアで使用

---

8種類のAndroidアプリテンプレート（Material3テーマ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose DynamicColor](https://zenn.dev/myougatheaxo/articles/android-compose-compose-dynamic-color-2026)
- [Compose DarkTheme](https://zenn.dev/myougatheaxo/articles/android-compose-compose-dark-theme-2026)
- [Compose Typography](https://zenn.dev/myougatheaxo/articles/android-compose-compose-typography-2026)
