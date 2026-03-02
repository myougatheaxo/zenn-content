---
title: "Typography完全ガイド — カスタムフォント/Material3 Typography/テキストスタイル"
emoji: "🔤"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "design"]
published: true
---

## この記事で学べること

**Typography**（カスタムフォント、Material3 Typography、テキストスタイル、可変フォント）を解説します。

---

## カスタムフォント

```kotlin
val NotoSansJP = FontFamily(
    Font(R.font.noto_sans_jp_regular, FontWeight.Normal),
    Font(R.font.noto_sans_jp_medium, FontWeight.Medium),
    Font(R.font.noto_sans_jp_bold, FontWeight.Bold),
    Font(R.font.noto_sans_jp_light, FontWeight.Light)
)

val AppTypography = Typography(
    displayLarge = TextStyle(fontFamily = NotoSansJP, fontWeight = FontWeight.Bold, fontSize = 57.sp),
    headlineLarge = TextStyle(fontFamily = NotoSansJP, fontWeight = FontWeight.Bold, fontSize = 32.sp),
    headlineMedium = TextStyle(fontFamily = NotoSansJP, fontWeight = FontWeight.Medium, fontSize = 28.sp),
    titleLarge = TextStyle(fontFamily = NotoSansJP, fontWeight = FontWeight.Medium, fontSize = 22.sp),
    titleMedium = TextStyle(fontFamily = NotoSansJP, fontWeight = FontWeight.Medium, fontSize = 16.sp),
    bodyLarge = TextStyle(fontFamily = NotoSansJP, fontWeight = FontWeight.Normal, fontSize = 16.sp, lineHeight = 24.sp),
    bodyMedium = TextStyle(fontFamily = NotoSansJP, fontWeight = FontWeight.Normal, fontSize = 14.sp, lineHeight = 20.sp),
    labelLarge = TextStyle(fontFamily = NotoSansJP, fontWeight = FontWeight.Medium, fontSize = 14.sp),
    labelSmall = TextStyle(fontFamily = NotoSansJP, fontWeight = FontWeight.Normal, fontSize = 11.sp)
)

@Composable
fun AppTheme(content: @Composable () -> Unit) {
    MaterialTheme(typography = AppTypography) { content() }
}
```

---

## テキストスタイリング

```kotlin
@Composable
fun StyledTextExamples() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(8.dp)) {
        Text("Display Large", style = MaterialTheme.typography.displayLarge)
        Text("Headline Medium", style = MaterialTheme.typography.headlineMedium)
        Text("Title Large", style = MaterialTheme.typography.titleLarge)
        Text("Body Large", style = MaterialTheme.typography.bodyLarge)

        // カスタムスタイル
        Text(
            "カスタムスタイル",
            style = MaterialTheme.typography.bodyLarge.copy(
                letterSpacing = 2.sp,
                textDecoration = TextDecoration.Underline,
                shadow = Shadow(Color.Gray, Offset(2f, 2f), blurRadius = 4f)
            )
        )

        // グラデーションテキスト
        Text(
            "グラデーション",
            style = MaterialTheme.typography.headlineMedium.copy(
                brush = Brush.linearGradient(listOf(Color(0xFF6200EE), Color(0xFF03DAC5)))
            )
        )
    }
}
```

---

## Google Fonts

```kotlin
val provider = GoogleFont.Provider(
    providerAuthority = "com.google.android.gms.fonts",
    providerPackage = "com.google.android.gms",
    certificates = R.array.com_google_android_gms_fonts_certs
)

val RobotoFont = GoogleFont("Roboto")

val RobotoFamily = FontFamily(
    Font(googleFont = RobotoFont, fontProvider = provider, weight = FontWeight.Normal),
    Font(googleFont = RobotoFont, fontProvider = provider, weight = FontWeight.Bold)
)
```

---

## まとめ

| レベル | スタイル |
|--------|---------|
| Display | 大見出し（57sp） |
| Headline | 見出し（28-32sp） |
| Title | タイトル（16-22sp） |
| Body | 本文（14-16sp） |
| Label | ラベル（11-14sp） |

- `FontFamily`でカスタムフォントを定義
- `Typography`でMaterial3スタイルを統一
- `Brush`でグラデーションテキスト
- Google Fontsでダウンロード可能フォント

---

8種類のAndroidアプリテンプレート（Typography設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Material3テーマ](https://zenn.dev/myougatheaxo/articles/android-compose-material3-theme-2026)
- [動的テーマ](https://zenn.dev/myougatheaxo/articles/android-compose-dynamic-theme-2026)
- [テキスト選択](https://zenn.dev/myougatheaxo/articles/android-compose-text-selection-2026)
