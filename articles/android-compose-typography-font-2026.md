---
title: "Typography/フォントガイド — カスタムフォント/GoogleFonts/動的サイズ"
emoji: "🔤"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "typography"]
published: true
---

## この記事で学べること

Composeでの**Typography設定**（カスタムフォント、Google Fonts、動的フォントサイズ）を解説します。

---

## カスタムフォントの設定

```kotlin
// res/font/ にフォントファイルを配置
val NotoSansJP = FontFamily(
    Font(R.font.notosansjp_regular, FontWeight.Normal),
    Font(R.font.notosansjp_medium, FontWeight.Medium),
    Font(R.font.notosansjp_bold, FontWeight.Bold)
)

val AppTypography = Typography(
    displayLarge = TextStyle(
        fontFamily = NotoSansJP,
        fontWeight = FontWeight.Bold,
        fontSize = 57.sp,
        lineHeight = 64.sp
    ),
    headlineMedium = TextStyle(
        fontFamily = NotoSansJP,
        fontWeight = FontWeight.Bold,
        fontSize = 28.sp,
        lineHeight = 36.sp
    ),
    titleLarge = TextStyle(
        fontFamily = NotoSansJP,
        fontWeight = FontWeight.Medium,
        fontSize = 22.sp,
        lineHeight = 28.sp
    ),
    bodyLarge = TextStyle(
        fontFamily = NotoSansJP,
        fontWeight = FontWeight.Normal,
        fontSize = 16.sp,
        lineHeight = 24.sp
    ),
    bodyMedium = TextStyle(
        fontFamily = NotoSansJP,
        fontWeight = FontWeight.Normal,
        fontSize = 14.sp,
        lineHeight = 20.sp
    ),
    labelLarge = TextStyle(
        fontFamily = NotoSansJP,
        fontWeight = FontWeight.Medium,
        fontSize = 14.sp,
        lineHeight = 20.sp
    )
)

// テーマに適用
@Composable
fun AppTheme(content: @Composable () -> Unit) {
    MaterialTheme(
        typography = AppTypography,
        content = content
    )
}
```

---

## Google Fonts Provider

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.compose.ui:ui-text-google-fonts:1.7.6")
}

val provider = GoogleFont.Provider(
    providerAuthority = "com.google.android.gms.fonts",
    providerPackage = "com.google.android.gms",
    certificates = R.array.com_google_android_gms_fonts_certs
)

val NotoSansJP = GoogleFont("Noto Sans JP")

val NotoSansFontFamily = FontFamily(
    Font(
        googleFont = NotoSansJP,
        fontProvider = provider,
        weight = FontWeight.Normal
    ),
    Font(
        googleFont = NotoSansJP,
        fontProvider = provider,
        weight = FontWeight.Medium
    ),
    Font(
        googleFont = NotoSansJP,
        fontProvider = provider,
        weight = FontWeight.Bold
    )
)
```

---

## テキストスタイルの使い分け

```kotlin
@Composable
fun TypographyShowcase() {
    Column(Modifier.padding(16.dp)) {
        Text("Display Large", style = MaterialTheme.typography.displayLarge)
        Text("Headline Medium", style = MaterialTheme.typography.headlineMedium)
        Text("Title Large", style = MaterialTheme.typography.titleLarge)
        Text("Body Large", style = MaterialTheme.typography.bodyLarge)
        Text("Body Medium", style = MaterialTheme.typography.bodyMedium)
        Text("Label Large", style = MaterialTheme.typography.labelLarge)
    }
}

// カスタムスタイルの拡張
@Composable
fun CustomText() {
    Text(
        "カスタムスタイル",
        style = MaterialTheme.typography.bodyLarge.copy(
            letterSpacing = 2.sp,
            lineHeight = 28.sp
        )
    )
}
```

---

## 動的フォントサイズ

```kotlin
@Composable
fun DynamicFontSize(
    baseFontSize: Int = 16,
    content: @Composable () -> Unit
) {
    val scaledTypography = Typography(
        bodyLarge = MaterialTheme.typography.bodyLarge.copy(
            fontSize = baseFontSize.sp
        ),
        bodyMedium = MaterialTheme.typography.bodyMedium.copy(
            fontSize = (baseFontSize - 2).sp
        ),
        titleLarge = MaterialTheme.typography.titleLarge.copy(
            fontSize = (baseFontSize + 6).sp
        )
    )

    MaterialTheme(typography = scaledTypography) {
        content()
    }
}

// 設定画面と連携
@Composable
fun AppWithFontSize() {
    val fontSize by settingsViewModel.fontSize.collectAsStateWithLifecycle()

    DynamicFontSize(baseFontSize = fontSize) {
        AppContent()
    }
}
```

---

## まとめ

- `FontFamily`でカスタムフォント定義
- `Typography`で全テキストスタイルを統一
- Google Fonts Providerでダウンロードフォント
- `MaterialTheme.typography.*`で一貫したスタイル参照
- `TextStyle.copy()`でスタイルのカスタマイズ
- 動的フォントサイズで設定画面と連携

---

8種類のAndroidアプリテンプレート（Typography設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [テキスト装飾ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-text-styling-2026)
- [カスタムテーマガイド](https://zenn.dev/myougatheaxo/articles/android-compose-theme-custom-2026)
- [ダークモード対応](https://zenn.dev/myougatheaxo/articles/android-dark-mode-guide-2026)
