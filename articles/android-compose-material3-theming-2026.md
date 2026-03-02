---
title: "Material3テーマカスタマイズ完全ガイド — ColorScheme/Typography/Shape"
emoji: "🎨"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**Material3**のテーマカスタマイズ（ColorScheme、Typography、Shapes、Dynamic Color）を解説します。

---

## カスタムColorScheme

```kotlin
private val LightColors = lightColorScheme(
    primary = Color(0xFF6750A4),
    onPrimary = Color.White,
    primaryContainer = Color(0xFFEADDFF),
    onPrimaryContainer = Color(0xFF21005D),
    secondary = Color(0xFF625B71),
    onSecondary = Color.White,
    secondaryContainer = Color(0xFFE8DEF8),
    tertiary = Color(0xFF7D5260),
    background = Color(0xFFFFFBFE),
    surface = Color(0xFFFFFBFE),
    error = Color(0xFFB3261E),
    onError = Color.White
)

private val DarkColors = darkColorScheme(
    primary = Color(0xFFD0BCFF),
    onPrimary = Color(0xFF381E72),
    primaryContainer = Color(0xFF4F378B),
    onPrimaryContainer = Color(0xFFEADDFF),
    secondary = Color(0xFFCCC2DC),
    background = Color(0xFF1C1B1F),
    surface = Color(0xFF1C1B1F)
)
```

---

## カスタムTypography

```kotlin
private val AppTypography = Typography(
    displayLarge = TextStyle(
        fontFamily = FontFamily(Font(R.font.noto_sans_jp_bold)),
        fontWeight = FontWeight.Bold,
        fontSize = 57.sp,
        lineHeight = 64.sp
    ),
    headlineMedium = TextStyle(
        fontFamily = FontFamily(Font(R.font.noto_sans_jp_medium)),
        fontWeight = FontWeight.Medium,
        fontSize = 28.sp,
        lineHeight = 36.sp
    ),
    titleLarge = TextStyle(
        fontFamily = FontFamily(Font(R.font.noto_sans_jp_medium)),
        fontSize = 22.sp,
        lineHeight = 28.sp
    ),
    bodyLarge = TextStyle(
        fontFamily = FontFamily(Font(R.font.noto_sans_jp_regular)),
        fontSize = 16.sp,
        lineHeight = 24.sp
    ),
    bodyMedium = TextStyle(
        fontFamily = FontFamily(Font(R.font.noto_sans_jp_regular)),
        fontSize = 14.sp,
        lineHeight = 20.sp
    ),
    labelLarge = TextStyle(
        fontFamily = FontFamily(Font(R.font.noto_sans_jp_medium)),
        fontWeight = FontWeight.Medium,
        fontSize = 14.sp,
        lineHeight = 20.sp
    )
)
```

---

## カスタムShapes

```kotlin
private val AppShapes = Shapes(
    extraSmall = RoundedCornerShape(4.dp),
    small = RoundedCornerShape(8.dp),
    medium = RoundedCornerShape(12.dp),
    large = RoundedCornerShape(16.dp),
    extraLarge = RoundedCornerShape(28.dp)
)
```

---

## テーマ統合

```kotlin
@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true,
    content: @Composable () -> Unit
) {
    val colorScheme = when {
        // Dynamic Color (Android 12+)
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

// 使用
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            AppTheme {
                Surface(color = MaterialTheme.colorScheme.background) {
                    AppNavigation()
                }
            }
        }
    }
}
```

---

## テーマ切替（手動）

```kotlin
@HiltViewModel
class ThemeViewModel @Inject constructor(
    private val dataStore: DataStore<Preferences>
) : ViewModel() {

    val themeMode = dataStore.data
        .map { prefs -> prefs[THEME_KEY] ?: "system" }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), "system")

    fun setTheme(mode: String) {
        viewModelScope.launch {
            dataStore.edit { it[THEME_KEY] = mode }
        }
    }

    companion object {
        private val THEME_KEY = stringPreferencesKey("theme_mode")
    }
}

@Composable
fun AppThemeWithPreference(
    themeViewModel: ThemeViewModel = hiltViewModel(),
    content: @Composable () -> Unit
) {
    val themeMode by themeViewModel.themeMode.collectAsStateWithLifecycle()

    val darkTheme = when (themeMode) {
        "dark" -> true
        "light" -> false
        else -> isSystemInDarkTheme()
    }

    AppTheme(darkTheme = darkTheme, content = content)
}
```

---

## カスタムカラー拡張

```kotlin
// Material3にない独自カラー
@Immutable
data class ExtendedColors(
    val success: Color,
    val warning: Color,
    val info: Color
)

val LocalExtendedColors = staticCompositionLocalOf {
    ExtendedColors(
        success = Color(0xFF4CAF50),
        warning = Color(0xFFFF9800),
        info = Color(0xFF2196F3)
    )
}

@Composable
fun AppTheme(content: @Composable () -> Unit) {
    CompositionLocalProvider(
        LocalExtendedColors provides ExtendedColors(
            success = Color(0xFF4CAF50),
            warning = Color(0xFFFF9800),
            info = Color(0xFF2196F3)
        )
    ) {
        MaterialTheme(content = content)
    }
}

// 使用
val colors = LocalExtendedColors.current
Icon(Icons.Default.Check, null, tint = colors.success)
```

---

## まとめ

- `lightColorScheme`/`darkColorScheme`でカラー定義
- `dynamicColorScheme`でAndroid 12+のDynamic Color
- `Typography`でフォントファミリー・サイズ一括設定
- `Shapes`で角丸サイズ統一
- DataStoreでテーマ設定永続化
- `CompositionLocalProvider`で独自カラー拡張

---

8種類のAndroidアプリテンプレート（M3テーマ設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Material3コンポーネント](https://zenn.dev/myougatheaxo/articles/android-compose-material3-components-2026)
- [カスタムComposable](https://zenn.dev/myougatheaxo/articles/android-compose-custom-composable-2026)
- [ダークモード](https://zenn.dev/myougatheaxo/articles/android-dark-mode-guide-2026)
