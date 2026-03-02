---
title: "Dynamic Color実装ガイド — Material You / Dynamic Colorでテーマを壁紙連動"
emoji: "🎨"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "materialyou"]
published: true
---

## この記事で学べること

Android 12+の**Dynamic Color（Material You）**で壁紙に連動するテーマの実装方法を解説します。

---

## Dynamic Colorの基本

```kotlin
@Composable
fun MyApp() {
    val dynamicColor = Build.VERSION.SDK_INT >= Build.VERSION_CODES.S
    val colorScheme = when {
        dynamicColor -> {
            val context = LocalContext.current
            if (isSystemInDarkTheme()) dynamicDarkColorScheme(context)
            else dynamicLightColorScheme(context)
        }
        isSystemInDarkTheme() -> darkColorScheme()
        else -> lightColorScheme()
    }

    MaterialTheme(
        colorScheme = colorScheme,
        typography = Typography
    ) {
        Surface(color = MaterialTheme.colorScheme.background) {
            MainScreen()
        }
    }
}
```

---

## カスタムカラー + Dynamic Color フォールバック

```kotlin
private val LightColors = lightColorScheme(
    primary = Color(0xFF6200EE),
    onPrimary = Color.White,
    secondary = Color(0xFF03DAC6),
    tertiary = Color(0xFFBB86FC),
    background = Color(0xFFFFFBFE),
    surface = Color(0xFFFFFBFE),
    error = Color(0xFFB00020)
)

private val DarkColors = darkColorScheme(
    primary = Color(0xFFBB86FC),
    onPrimary = Color.Black,
    secondary = Color(0xFF03DAC6),
    tertiary = Color(0xFF6200EE),
    background = Color(0xFF121212),
    surface = Color(0xFF121212),
    error = Color(0xFFCF6679)
)

@Composable
fun MyAppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true,
    content: @Composable () -> Unit
) {
    val colorScheme = when {
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            val context = LocalContext.current
            if (darkTheme) dynamicDarkColorScheme(context)
            else dynamicLightColorScheme(context)
        }
        darkTheme -> DarkColors
        else -> LightColors
    }

    MaterialTheme(
        colorScheme = colorScheme,
        content = content
    )
}
```

---

## テーマカラーの活用

```kotlin
@Composable
fun ThemedComponents() {
    Column(Modifier.padding(16.dp)) {
        // Primary
        Card(
            colors = CardDefaults.cardColors(
                containerColor = MaterialTheme.colorScheme.primaryContainer
            )
        ) {
            Text(
                "Primary Container",
                Modifier.padding(16.dp),
                color = MaterialTheme.colorScheme.onPrimaryContainer
            )
        }

        Spacer(Modifier.height(8.dp))

        // Secondary
        Card(
            colors = CardDefaults.cardColors(
                containerColor = MaterialTheme.colorScheme.secondaryContainer
            )
        ) {
            Text(
                "Secondary Container",
                Modifier.padding(16.dp),
                color = MaterialTheme.colorScheme.onSecondaryContainer
            )
        }

        Spacer(Modifier.height(8.dp))

        // Tertiary
        Card(
            colors = CardDefaults.cardColors(
                containerColor = MaterialTheme.colorScheme.tertiaryContainer
            )
        ) {
            Text(
                "Tertiary Container",
                Modifier.padding(16.dp),
                color = MaterialTheme.colorScheme.onTertiaryContainer
            )
        }

        Spacer(Modifier.height(16.dp))

        // Surface variants
        Surface(
            color = MaterialTheme.colorScheme.surfaceVariant,
            shape = RoundedCornerShape(12.dp)
        ) {
            Text(
                "Surface Variant",
                Modifier.padding(16.dp),
                color = MaterialTheme.colorScheme.onSurfaceVariant
            )
        }
    }
}
```

---

## テーマ切り替えUI

```kotlin
@Composable
fun ThemeSettingsScreen(
    isDynamic: Boolean,
    isDarkMode: Boolean,
    onDynamicChange: (Boolean) -> Unit,
    onDarkModeChange: (Boolean) -> Unit
) {
    LazyColumn {
        item {
            ListItem(
                headlineContent = { Text("ダークモード") },
                trailingContent = {
                    Switch(checked = isDarkMode, onCheckedChange = onDarkModeChange)
                }
            )
        }

        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
            item {
                ListItem(
                    headlineContent = { Text("Dynamic Color") },
                    supportingContent = { Text("壁紙に合わせたカラーテーマ") },
                    trailingContent = {
                        Switch(checked = isDynamic, onCheckedChange = onDynamicChange)
                    }
                )
            }
        }

        item {
            Text(
                "カラープレビュー",
                Modifier.padding(16.dp),
                style = MaterialTheme.typography.titleMedium
            )
        }

        item {
            Row(
                Modifier.fillMaxWidth().padding(16.dp),
                horizontalArrangement = Arrangement.spacedBy(8.dp)
            ) {
                listOf(
                    "Primary" to MaterialTheme.colorScheme.primary,
                    "Secondary" to MaterialTheme.colorScheme.secondary,
                    "Tertiary" to MaterialTheme.colorScheme.tertiary
                ).forEach { (name, color) ->
                    Column(
                        Modifier.weight(1f),
                        horizontalAlignment = Alignment.CenterHorizontally
                    ) {
                        Box(
                            Modifier
                                .size(48.dp)
                                .clip(CircleShape)
                                .background(color)
                        )
                        Text(name, style = MaterialTheme.typography.labelSmall)
                    }
                }
            }
        }
    }
}
```

---

## まとめ

- `dynamicLightColorScheme()`/`dynamicDarkColorScheme()`でDynamic Color
- Android 12未満はカスタムカラーにフォールバック
- `MaterialTheme.colorScheme`でprimary/secondary/tertiary + Container
- `surfaceVariant`/`outline`で微妙な色分け
- テーマ設定UIでDynamic Color/ダークモードを切り替え可能に

---

8種類のAndroidアプリテンプレート（Dynamic Color対応設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [カスタムテーマ完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-theme-custom-2026)
- [ダークモード対応ガイド](https://zenn.dev/myougatheaxo/articles/android-dark-mode-guide-2026)
- [Material3コンポーネント一覧](https://zenn.dev/myougatheaxo/articles/compose-material3-components-2026)
