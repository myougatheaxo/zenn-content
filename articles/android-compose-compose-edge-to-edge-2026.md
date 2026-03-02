---
title: "Edge-to-Edge完全ガイド — enableEdgeToEdge/WindowInsets/SystemBars"
emoji: "📱"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**Edge-to-Edge**（enableEdgeToEdge、WindowInsets、SystemBars、ステータスバー/ナビバー制御）を解説します。

---

## enableEdgeToEdge

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        enableEdgeToEdge()
        super.onCreate(savedInstanceState)
        setContent {
            MyTheme {
                Scaffold(
                    modifier = Modifier.fillMaxSize(),
                    contentWindowInsets = WindowInsets(0)
                ) { padding ->
                    MainContent(Modifier.padding(padding))
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
fun WindowInsetsExample() {
    Column(
        Modifier
            .fillMaxSize()
            .windowInsetsPadding(WindowInsets.systemBars)
    ) {
        // ステータスバー分のパディング自動適用
        Text("コンテンツ", Modifier.padding(16.dp))
    }
}

@Composable
fun SelectiveInsets() {
    Column(Modifier.fillMaxSize()) {
        // ステータスバーのみ
        Box(
            Modifier
                .fillMaxWidth()
                .windowInsetsPadding(WindowInsets.statusBars)
                .background(MaterialTheme.colorScheme.primary)
        ) {
            Text("ヘッダー", Modifier.padding(16.dp), color = Color.White)
        }

        // コンテンツ
        Box(Modifier.weight(1f).padding(16.dp)) {
            Text("メインコンテンツ")
        }

        // ナビバーのみ
        Box(
            Modifier
                .fillMaxWidth()
                .windowInsetsPadding(WindowInsets.navigationBars)
                .background(MaterialTheme.colorScheme.surface)
        ) {
            Text("フッター", Modifier.padding(16.dp))
        }
    }
}
```

---

## 個別Insets

```kotlin
@Composable
fun InsetsTypes() {
    val statusBarHeight = WindowInsets.statusBars.asPaddingValues()
        .calculateTopPadding()
    val navBarHeight = WindowInsets.navigationBars.asPaddingValues()
        .calculateBottomPadding()
    val imeHeight = WindowInsets.ime.asPaddingValues()
        .calculateBottomPadding()

    Column(Modifier.padding(16.dp)) {
        Text("ステータスバー: $statusBarHeight")
        Text("ナビゲーションバー: $navBarHeight")
        Text("キーボード: $imeHeight")
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `enableEdgeToEdge()` | 全画面表示有効化 |
| `WindowInsets.systemBars` | 全システムバー |
| `WindowInsets.statusBars` | ステータスバー |
| `WindowInsets.navigationBars` | ナビバー |

- `enableEdgeToEdge()`でAndroid 15+のEdge-to-Edge対応
- `windowInsetsPadding`で個別にインセット適用
- `Scaffold`は内部でインセットを自動処理
- `WindowInsets.ime`でキーボード分のインセット

---

8種類のAndroidアプリテンプレート（Edge-to-Edge対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Scaffold](https://zenn.dev/myougatheaxo/articles/android-compose-compose-scaffold-2026)
- [キーボード/IME](https://zenn.dev/myougatheaxo/articles/android-compose-compose-keyboard-ime-2026)
- [ダイナミックテーマ](https://zenn.dev/myougatheaxo/articles/android-compose-dynamic-theme-2026)
