---
title: "Compose StatusBar完全ガイド — SystemBarColor/EdgeToEdge/WindowInsets/透過ステータスバー"
emoji: "📱"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "systemui"]
published: true
---

## この記事で学べること

**Compose StatusBar**（Edge-to-Edge、WindowInsets、ステータスバー色制御、透過表示）を解説します。

---

## Edge-to-Edge設定

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge() // Android 15+標準

        setContent {
            MyTheme {
                Scaffold(
                    modifier = Modifier.fillMaxSize()
                ) { innerPadding ->
                    MainContent(Modifier.padding(innerPadding))
                }
            }
        }
    }
}
```

---

## WindowInsets制御

```kotlin
@Composable
fun EdgeToEdgeScreen() {
    Box(
        Modifier
            .fillMaxSize()
            .background(MaterialTheme.colorScheme.primaryContainer)
    ) {
        // ステータスバー下にコンテンツ
        Column(
            Modifier
                .fillMaxSize()
                .statusBarsPadding() // ステータスバー分のパディング
                .navigationBarsPadding() // ナビバー分のパディング
        ) {
            Text("Edge-to-Edge対応", style = MaterialTheme.typography.headlineMedium)
        }
    }
}

@Composable
fun InsetsExample() {
    val statusBarHeight = WindowInsets.statusBars
        .asPaddingValues().calculateTopPadding()
    val navBarHeight = WindowInsets.navigationBars
        .asPaddingValues().calculateBottomPadding()

    Column(Modifier.fillMaxSize()) {
        Spacer(Modifier.height(statusBarHeight))
        Text("ステータスバー高さ: $statusBarHeight")
        Spacer(Modifier.weight(1f))
        Text("ナビバー高さ: $navBarHeight")
        Spacer(Modifier.height(navBarHeight))
    }
}
```

---

## ステータスバー色制御

```kotlin
@Composable
fun DynamicStatusBar() {
    val scrollState = rememberScrollState()
    val isScrolled = scrollState.value > 100

    // スクロールに応じてステータスバースタイル変更
    val view = LocalView.current
    LaunchedEffect(isScrolled) {
        val window = (view.context as Activity).window
        WindowCompat.getInsetsController(window, view).apply {
            isAppearanceLightStatusBars = isScrolled // trueで暗いアイコン
        }
    }

    Column(Modifier.fillMaxSize().verticalScroll(scrollState)) {
        // ヒーロー画像（暗い背景→明るいアイコン）
        Box(Modifier.fillMaxWidth().height(300.dp).background(Color.DarkGray)) {
            Text("ヒーロー", color = Color.White, modifier = Modifier.align(Alignment.Center))
        }
        // コンテンツ（明るい背景→暗いアイコン）
        Column(Modifier.padding(16.dp)) {
            repeat(20) {
                Text("コンテンツ $it", Modifier.padding(vertical = 8.dp))
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `enableEdgeToEdge` | Edge-to-Edge有効化 |
| `statusBarsPadding` | ステータスバーパディング |
| `WindowInsets` | システムバー情報取得 |
| `isAppearanceLightStatusBars` | アイコン色制御 |

- `enableEdgeToEdge()`でAndroid 15+のEdge-to-Edge対応
- `statusBarsPadding()`/`navigationBarsPadding()`で安全なパディング
- `WindowInsets`でシステムバーのサイズを取得
- `isAppearanceLightStatusBars`でアイコン色を動的に切り替え

---

8種類のAndroidアプリテンプレート（Edge-to-Edge対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose ImmersiveMode](https://zenn.dev/myougatheaxo/articles/android-compose-compose-immersive-mode-2026)
- [Compose Scaffold](https://zenn.dev/myougatheaxo/articles/android-compose-compose-scaffold-snackbar-2026)
- [Compose TopAppBar](https://zenn.dev/myougatheaxo/articles/android-compose-compose-topappbar-2026)
