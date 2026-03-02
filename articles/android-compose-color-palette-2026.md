---
title: "カラーパレット/テーマ拡張完全ガイド — Dynamic Color/カスタムカラー/ダークモード"
emoji: "🌈"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "design"]
published: true
---

## この記事で学べること

**カラーパレット/テーマ拡張**（Dynamic Color、カスタムカラーロール、画像からの色抽出、ダークモード対応）を解説します。

---

## Dynamic Color

```kotlin
@Composable
fun AppTheme(
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
        darkTheme -> darkColorScheme(
            primary = Color(0xFFBB86FC),
            secondary = Color(0xFF03DAC5),
            background = Color(0xFF121212)
        )
        else -> lightColorScheme(
            primary = Color(0xFF6200EE),
            secondary = Color(0xFF03DAC5),
            background = Color(0xFFFFFBFE)
        )
    }

    MaterialTheme(colorScheme = colorScheme, content = content)
}
```

---

## カスタムカラーロール

```kotlin
// MaterialTheme.colorScheme に含まれないカスタムカラー
data class ExtendedColors(
    val success: Color,
    val onSuccess: Color,
    val successContainer: Color,
    val warning: Color,
    val onWarning: Color,
    val warningContainer: Color,
    val info: Color,
    val onInfo: Color
)

val LocalExtendedColors = staticCompositionLocalOf {
    ExtendedColors(
        success = Color(0xFF4CAF50),
        onSuccess = Color.White,
        successContainer = Color(0xFFC8E6C9),
        warning = Color(0xFFFF9800),
        onWarning = Color.White,
        warningContainer = Color(0xFFFFE0B2),
        info = Color(0xFF2196F3),
        onInfo = Color.White
    )
}

// テーマに統合
@Composable
fun AppTheme(content: @Composable () -> Unit) {
    val extendedColors = if (isSystemInDarkTheme()) {
        ExtendedColors(
            success = Color(0xFF81C784),
            onSuccess = Color(0xFF1B5E20),
            successContainer = Color(0xFF2E7D32),
            warning = Color(0xFFFFB74D),
            onWarning = Color(0xFFE65100),
            warningContainer = Color(0xFFEF6C00),
            info = Color(0xFF64B5F6),
            onInfo = Color(0xFF0D47A1)
        )
    } else {
        ExtendedColors(/* ライトテーマの値 */)
    }

    CompositionLocalProvider(LocalExtendedColors provides extendedColors) {
        MaterialTheme(content = content)
    }
}

// 使用
val MaterialTheme.extendedColors: ExtendedColors
    @Composable @ReadOnlyComposable
    get() = LocalExtendedColors.current

@Composable
fun SuccessBanner(message: String) {
    Surface(color = MaterialTheme.extendedColors.successContainer) {
        Text(message, color = MaterialTheme.extendedColors.success)
    }
}
```

---

## 画像から色抽出

```kotlin
@Composable
fun ImageWithExtractedColors(imageUrl: String) {
    var dominantColor by remember { mutableStateOf(Color.Transparent) }

    val painter = rememberAsyncImagePainter(
        ImageRequest.Builder(LocalContext.current)
            .data(imageUrl)
            .allowHardware(false) // Palette用
            .build()
    )

    LaunchedEffect(painter.state) {
        val state = painter.state
        if (state is AsyncImagePainter.State.Success) {
            val bitmap = (state.result.drawable as BitmapDrawable).bitmap
            val palette = Palette.from(bitmap).generate()
            dominantColor = Color(palette.getDominantColor(0))
        }
    }

    Column {
        Image(painter = painter, contentDescription = null, modifier = Modifier.height(200.dp))
        Surface(color = dominantColor, modifier = Modifier.fillMaxWidth().height(4.dp)) {}
    }
}
```

---

## まとめ

| 手法 | 用途 |
|------|------|
| `dynamicColorScheme` | 壁紙連動カラー |
| `CompositionLocal` | カスタムカラーロール |
| `Palette` | 画像から色抽出 |
| `darkColorScheme` | ダークモード |

- Dynamic Colorで端末の壁紙に連動したテーマ
- `CompositionLocal`でsuccess/warning等を追加
- `Palette`で画像からドミナントカラーを抽出
- ライト/ダーク両方のカラー定義を忘れずに

---

8種類のAndroidアプリテンプレート（テーマカスタマイズ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Material3テーマ](https://zenn.dev/myougatheaxo/articles/android-compose-material3-theme-2026)
- [CompositionLocal](https://zenn.dev/myougatheaxo/articles/android-compose-compositionlocal-2026)
- [ダークモード](https://zenn.dev/myougatheaxo/articles/android-compose-dark-theme-2026)
