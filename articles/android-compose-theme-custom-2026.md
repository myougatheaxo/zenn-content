---
title: "Composeカスタムテーマ設計 — ブランドカラー・フォント・Shapeの統一"
emoji: "🎭"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "design"]
published: true
---

## この記事で学べること

Material3のカラー・タイポグラフィ・シェイプを**自分のブランドに合わせてカスタマイズ**する方法を解説します。

---

## カスタムカラースキーム

```kotlin
private val LightColors = lightColorScheme(
    primary = Color(0xFF1B5E20),          // 深緑
    onPrimary = Color.White,
    primaryContainer = Color(0xFFA5D6A7),
    secondary = Color(0xFFFF6F00),         // オレンジ
    onSecondary = Color.White,
    background = Color(0xFFFAFAFA),
    surface = Color.White,
    error = Color(0xFFD32F2F)
)

private val DarkColors = darkColorScheme(
    primary = Color(0xFF81C784),
    onPrimary = Color(0xFF003300),
    primaryContainer = Color(0xFF1B5E20),
    secondary = Color(0xFFFFB74D),
    onSecondary = Color(0xFF3E2723),
    background = Color(0xFF121212),
    surface = Color(0xFF1E1E1E),
    error = Color(0xFFEF9A9A)
)
```

---

## カスタムタイポグラフィ

```kotlin
val AppTypography = Typography(
    headlineLarge = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Bold,
        fontSize = 28.sp,
        lineHeight = 36.sp
    ),
    titleMedium = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.SemiBold,
        fontSize = 18.sp,
        lineHeight = 24.sp
    ),
    bodyLarge = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Normal,
        fontSize = 16.sp,
        lineHeight = 24.sp
    ),
    labelMedium = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Medium,
        fontSize = 12.sp,
        lineHeight = 16.sp
    )
)
```

---

## カスタムシェイプ

```kotlin
val AppShapes = Shapes(
    small = RoundedCornerShape(4.dp),
    medium = RoundedCornerShape(12.dp),
    large = RoundedCornerShape(16.dp),
    extraLarge = RoundedCornerShape(24.dp)
)
```

---

## テーマの組み立て

```kotlin
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
        typography = AppTypography,
        shapes = AppShapes,
        content = content
    )
}
```

---

## カスタムプロパティの追加

```kotlin
// アプリ固有のカラーを追加
data class ExtendedColors(
    val success: Color,
    val warning: Color,
    val info: Color
)

val LocalExtendedColors = staticCompositionLocalOf {
    ExtendedColors(
        success = Color(0xFF4CAF50),
        warning = Color(0xFFFFC107),
        info = Color(0xFF2196F3)
    )
}

@Composable
fun MyAppTheme(content: @Composable () -> Unit) {
    val extendedColors = ExtendedColors(
        success = Color(0xFF4CAF50),
        warning = Color(0xFFFFC107),
        info = Color(0xFF2196F3)
    )

    CompositionLocalProvider(
        LocalExtendedColors provides extendedColors
    ) {
        MaterialTheme(/* ... */) {
            content()
        }
    }
}

// 使い方
val successColor = LocalExtendedColors.current.success
```

---

## Material Theme Builderの活用

Googleの**Material Theme Builder**（m3.material.io）でカラースキームを自動生成できます。

1. ブランドカラーを入力
2. ライト/ダーク両方のカラーが自動生成
3. Kotlin/Compose形式でエクスポート

---

## テーマの使い方ガイド

```kotlin
@Composable
fun ThemedCard() {
    Card(
        colors = CardDefaults.cardColors(
            containerColor = MaterialTheme.colorScheme.surfaceVariant
        ),
        shape = MaterialTheme.shapes.medium
    ) {
        Text(
            "テーマ適用済み",
            style = MaterialTheme.typography.titleMedium,
            color = MaterialTheme.colorScheme.onSurfaceVariant
        )
    }
}
```

**直接Color値を使わず**、`MaterialTheme.colorScheme`経由で参照するのが鉄則。

---

## まとめ

- `lightColorScheme()`/`darkColorScheme()`でカラー定義
- `Typography()`でフォント設定
- `Shapes()`で角丸設定
- `CompositionLocalProvider`でカスタムプロパティ追加
- Material Theme Builderでカラー自動生成
- 常に`MaterialTheme.XXX`経由で値を参照

---

8種類のAndroidアプリテンプレート（カスタムテーマ設定済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Material3テーマ完全ガイド](https://zenn.dev/myougatheaxo/articles/material3-theme-guide-2026)
- [ダークモード完全対応ガイド](https://zenn.dev/myougatheaxo/articles/android-dark-mode-guide-2026)
- [Material3コンポーネントカタログ](https://zenn.dev/myougatheaxo/articles/compose-material3-components-2026)
