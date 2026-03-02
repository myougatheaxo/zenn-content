---
title: "Compose DynamicColor完全ガイド — 壁紙カラー抽出/harmonize/カスタムシード"
emoji: "🌈"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**Compose DynamicColor**（壁紙カラー抽出、harmonize、カスタムシードカラー、Material Youカラー生成）を解説します。

---

## 基本DynamicColor

```kotlin
@Composable
fun DynamicColorDemo() {
    val context = LocalContext.current
    val isDynamic = Build.VERSION.SDK_INT >= Build.VERSION_CODES.S

    val colorScheme = if (isDynamic) {
        dynamicLightColorScheme(context)
    } else {
        lightColorScheme()
    }

    MaterialTheme(colorScheme = colorScheme) {
        Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(8.dp)) {
            ColorSwatch("Primary", MaterialTheme.colorScheme.primary)
            ColorSwatch("Secondary", MaterialTheme.colorScheme.secondary)
            ColorSwatch("Tertiary", MaterialTheme.colorScheme.tertiary)
            ColorSwatch("PrimaryContainer", MaterialTheme.colorScheme.primaryContainer)
        }
    }
}

@Composable
fun ColorSwatch(name: String, color: Color) {
    Row(verticalAlignment = Alignment.CenterVertically) {
        Box(Modifier.size(40.dp).background(color, RoundedCornerShape(8.dp)))
        Spacer(Modifier.width(12.dp))
        Text(name)
    }
}
```

---

## harmonize（カスタムカラー調和）

```kotlin
@Composable
fun HarmonizedColors() {
    val context = LocalContext.current

    // ブランドカラーをDynamic Colorと調和させる
    val brandRed = Color(0xFFE91E63)
    val harmonized = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
        val harmonizedArgb = MaterialColors.harmonize(
            brandRed.toArgb(),
            MaterialTheme.colorScheme.primary.toArgb()
        )
        Color(harmonizedArgb)
    } else {
        brandRed
    }

    Card(colors = CardDefaults.cardColors(containerColor = harmonized.copy(alpha = 0.1f))) {
        Text(
            "調和されたブランドカラー",
            Modifier.padding(16.dp),
            color = harmonized
        )
    }
}
```

---

## テーマ切替UI

```kotlin
enum class ThemeMode { SYSTEM, LIGHT, DARK }

@Composable
fun ThemeSelector(current: ThemeMode, onSelect: (ThemeMode) -> Unit) {
    Column {
        Text("テーマ設定", style = MaterialTheme.typography.titleMedium)
        Spacer(Modifier.height(8.dp))
        ThemeMode.entries.forEach { mode ->
            Row(
                Modifier.fillMaxWidth().clickable { onSelect(mode) }.padding(12.dp),
                verticalAlignment = Alignment.CenterVertically
            ) {
                RadioButton(selected = current == mode, onClick = { onSelect(mode) })
                Spacer(Modifier.width(8.dp))
                Text(when (mode) {
                    ThemeMode.SYSTEM -> "システム設定に従う"
                    ThemeMode.LIGHT -> "ライト"
                    ThemeMode.DARK -> "ダーク"
                })
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `dynamicLightColorScheme` | 壁紙ライトカラー |
| `dynamicDarkColorScheme` | 壁紙ダークカラー |
| `MaterialColors.harmonize` | カラー調和 |
| `Build.VERSION_CODES.S` | API 31チェック |

- Dynamic ColorはAndroid 12 (API 31)以上で利用可能
- `harmonize`でブランドカラーを壁紙カラーに調和
- フォールバック用にカスタムカラースキームを必ず用意
- テーマ設定をDataStoreで永続化推奨

---

8種類のAndroidアプリテンプレート（Material3テーマ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose MaterialYou](https://zenn.dev/myougatheaxo/articles/android-compose-compose-material-you-2026)
- [Compose DarkTheme](https://zenn.dev/myougatheaxo/articles/android-compose-compose-dark-theme-2026)
- [Compose ColorScheme](https://zenn.dev/myougatheaxo/articles/android-compose-compose-color-scheme-2026)
