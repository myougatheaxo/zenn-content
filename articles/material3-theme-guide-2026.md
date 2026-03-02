---
title: "Material3テーマ完全ガイド — AIが生成するアプリの色を自由に変える"
emoji: "🎨"
type: "tech"
topics: ["android", "jetpackcompose", "material3", "kotlin", "design"]
published: true
---

## この記事で学べること

AIでAndroidアプリを生成すると、Material3テーマが自動で適用されます。でも「色を変えたい」「ダークモードに対応したい」と思ったとき、どこを触ればいいのか。

この記事では、**AIが生成するMaterial3テーマの構造と、カスタマイズ方法**を解説します。

---

## Material3の色システム

Material3では、**1つのseed colorから25色以上が自動生成**されます。

```kotlin
// ui/theme/Color.kt
val Purple80 = Color(0xFFD0BCFF)
val PurpleGrey80 = Color(0xFFCCC2DC)
val Pink80 = Color(0xFFEFB8C8)

val Purple40 = Color(0xFF6650a4)
val PurpleGrey40 = Color(0xFF625b71)
val Pink40 = Color(0xFF7D5260)
```

この6色を変えるだけで、アプリ全体の配色が変わります。

### ColorSchemeの構造

```kotlin
// ui/theme/Theme.kt
private val DarkColorScheme = darkColorScheme(
    primary = Purple80,
    secondary = PurpleGrey80,
    tertiary = Pink80
)

private val LightColorScheme = lightColorScheme(
    primary = Purple40,
    secondary = PurpleGrey40,
    tertiary = Pink40,
    background = Color(0xFFFFFBFE),
    surface = Color(0xFFFFFBFE),
    onPrimary = Color.White,
    onSecondary = Color.White,
    onTertiary = Color.White,
    onBackground = Color(0xFF1C1B1F),
    onSurface = Color(0xFF1C1B1F),
)
```

### 色を変える3ステップ

**Step 1**: [Material Theme Builder](https://m3.material.io/theme-builder)でseed colorを選ぶ

**Step 2**: Export → Jetpack Compose を選択

**Step 3**: 生成された`Color.kt`と`Theme.kt`をプロジェクトにコピー

これだけで全画面の配色が統一的に変わります。

---

## Dynamic Color（Android 12+）

Android 12以降では、ユーザーの壁紙から色を自動取得できます。

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
        darkTheme -> DarkColorScheme
        else -> LightColorScheme
    }

    MaterialTheme(
        colorScheme = colorScheme,
        typography = Typography,
        content = content
    )
}
```

`dynamicColor = true`にすると、ユーザーの壁紙に合わせてアプリの色が変わります。`false`にすれば固定カラー。

---

## ダークモード対応

AIが生成するコードは基本的にダークモード対応済みです。`isSystemInDarkTheme()`で自動切り替え。

手動トグルを追加する場合：

```kotlin
@Composable
fun SettingsScreen(viewModel: SettingsViewModel) {
    val isDark by viewModel.isDarkMode.collectAsState()

    Switch(
        checked = isDark,
        onCheckedChange = { viewModel.toggleDarkMode() }
    )
}
```

---

## Typography（フォント）

```kotlin
val Typography = Typography(
    bodyLarge = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Normal,
        fontSize = 16.sp,
        lineHeight = 24.sp,
    ),
    titleLarge = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Normal,
        fontSize = 22.sp,
        lineHeight = 28.sp,
    ),
    labelSmall = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Medium,
        fontSize = 11.sp,
        lineHeight = 16.sp,
    )
)
```

日本語フォントを使う場合は、`fontFamily`を`FontFamily(Font(R.font.noto_sans_jp))`に変更するだけ。

---

## AIが生成するテーマの特徴

Claude Codeで生成したアプリのテーマコードを分析すると：

| 項目 | 生成される内容 |
|------|--------------|
| Light/Dark | 両方自動生成 |
| Dynamic Color | Android 12+で自動有効 |
| Typography | Material3デフォルト |
| Shape | RoundedCornerShape(12.dp) |
| カラーロール | primary/secondary/tertiary完備 |

**人間が変更すべき箇所は2つだけ**：`Color.kt`の6色と、必要ならフォント。

---

## まとめ

- Material3の色は`Color.kt`の6色を変えるだけで全体が変わる
- Material Theme Builderで視覚的にカラー生成可能
- Dynamic Colorでユーザーの壁紙に自動適応
- ダークモードはAIが自動実装してくれる

---

8種類のAndroidアプリテンプレート（全てMaterial3対応、ダークモード完備）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [AIでAndroidアプリを47秒で作った話](https://zenn.dev/myougatheaxo/articles/ai-android-app-47seconds)
- [AIコーディングツール完全比較（7ツール）](https://zenn.dev/myougatheaxo/articles/ai-coding-tools-complete-2026)
- [Kotlin Coroutine入門](https://zenn.dev/myougatheaxo/articles/kotlin-coroutine-android-basics)
- [Gradle設定入門](https://zenn.dev/myougatheaxo/articles/android-gradle-basics-2026)
