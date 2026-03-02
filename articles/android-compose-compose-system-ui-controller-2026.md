---
title: "Compose SystemUIController完全ガイド — ステータスバー/ナビゲーションバー/enableEdgeToEdge"
emoji: "📱"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**Compose SystemUI制御**（ステータスバー色、ナビゲーションバー色、enableEdgeToEdge、透明バー）を解説します。

---

## enableEdgeToEdge（推奨）

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        enableEdgeToEdge()
        super.onCreate(savedInstanceState)
        setContent {
            AppTheme {
                Scaffold(
                    modifier = Modifier.fillMaxSize()
                ) { innerPadding ->
                    MainScreen(Modifier.padding(innerPadding))
                }
            }
        }
    }
}
```

---

## ステータスバー制御

```kotlin
@Composable
fun TransparentStatusBar() {
    val view = LocalView.current
    if (!view.isInEditMode) {
        SideEffect {
            val window = (view.context as Activity).window
            window.statusBarColor = android.graphics.Color.TRANSPARENT

            // アイコンの色（ライト/ダーク）
            WindowCompat.getInsetsController(window, view).apply {
                isAppearanceLightStatusBars = true // ダーク（黒）アイコン
            }
        }
    }
}

@Composable
fun DarkStatusBar() {
    val view = LocalView.current
    val darkTheme = isSystemInDarkTheme()

    SideEffect {
        val window = (view.context as Activity).window
        WindowCompat.getInsetsController(window, view).apply {
            isAppearanceLightStatusBars = !darkTheme
            isAppearanceLightNavigationBars = !darkTheme
        }
    }
}
```

---

## 画面別設定

```kotlin
@Composable
fun ImmersiveScreen() {
    val view = LocalView.current

    DisposableEffect(Unit) {
        val window = (view.context as Activity).window
        val controller = WindowCompat.getInsetsController(window, view)

        // 全画面表示
        controller.hide(WindowInsetsCompat.Type.systemBars())
        controller.systemBarsBehavior =
            WindowInsetsControllerCompat.BEHAVIOR_SHOW_TRANSIENT_BARS_BY_SWIPE

        onDispose {
            controller.show(WindowInsetsCompat.Type.systemBars())
        }
    }

    Box(
        Modifier.fillMaxSize().background(Color.Black),
        contentAlignment = Alignment.Center
    ) {
        Text("没入モード", color = Color.White, style = MaterialTheme.typography.headlineLarge)
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `enableEdgeToEdge()` | Edge-to-edge表示 |
| `isAppearanceLightStatusBars` | ステータスバーアイコン色 |
| `WindowInsetsController.hide` | システムバー非表示 |
| `BEHAVIOR_SHOW_TRANSIENT_BARS_BY_SWIPE` | スワイプ表示 |

- `enableEdgeToEdge()`が最新推奨API
- `SideEffect`/`DisposableEffect`でライフサイクル管理
- 画面別に没入モード/通常モードを切替
- ダークテーマに応じてアイコン色を自動切替

---

8種類のAndroidアプリテンプレート（Edge-to-Edge対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose EdgeToEdge](https://zenn.dev/myougatheaxo/articles/android-compose-compose-edge-to-edge-2026)
- [Compose DarkTheme](https://zenn.dev/myougatheaxo/articles/android-compose-compose-dark-theme-2026)
- [Compose WindowInsets](https://zenn.dev/myougatheaxo/articles/android-compose-compose-window-insets-consume-2026)
