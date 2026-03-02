---
title: "WindowInsets完全ガイド — ステータスバー/ナビバー/IME/SafeArea"
emoji: "📱"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "windowinsets"]
published: true
---

## この記事で学べること

**WindowInsets**（ステータスバー、ナビゲーションバー、IME（キーボード）、SafeArea、Edge-to-Edge連携）を解説します。

---

## WindowInsets種類

```kotlin
@Composable
fun InsetsExample() {
    val statusBars = WindowInsets.statusBars         // ステータスバー
    val navBars = WindowInsets.navigationBars         // ナビゲーションバー
    val ime = WindowInsets.ime                        // キーボード
    val systemBars = WindowInsets.systemBars          // statusBars + navBars
    val safeDrawing = WindowInsets.safeDrawing        // 全安全領域
    val displayCutout = WindowInsets.displayCutout    // ノッチ/パンチホール
}
```

---

## Modifier.windowInsetsPadding

```kotlin
@Composable
fun EdgeToEdgeScreen() {
    Column(
        Modifier
            .fillMaxSize()
            .windowInsetsPadding(WindowInsets.systemBars)
    ) {
        Text("ステータスバーの下", style = MaterialTheme.typography.headlineMedium)
        Spacer(Modifier.weight(1f))
        Text("ナビバーの上")
    }
}

// 個別指定
@Composable
fun IndividualInsets() {
    Column(Modifier.fillMaxSize()) {
        Spacer(Modifier.windowInsetsTopHeight(WindowInsets.statusBars))

        Column(Modifier.weight(1f).padding(horizontal = 16.dp)) {
            Text("コンテンツ")
        }

        Spacer(Modifier.windowInsetsBottomHeight(WindowInsets.navigationBars))
    }
}
```

---

## IME（キーボード）対応

```kotlin
@Composable
fun ChatInput() {
    var text by remember { mutableStateOf("") }

    Column(
        Modifier
            .fillMaxSize()
            .imePadding()  // キーボード表示時に自動パディング
    ) {
        LazyColumn(Modifier.weight(1f)) {
            // メッセージ一覧
        }

        Row(
            Modifier
                .fillMaxWidth()
                .navigationBarsPadding()  // ナビバー分のパディング
                .padding(8.dp)
        ) {
            OutlinedTextField(
                value = text,
                onValueChange = { text = it },
                modifier = Modifier.weight(1f),
                placeholder = { Text("メッセージ") }
            )
            IconButton(onClick = { /* 送信 */ }) {
                Icon(Icons.AutoMirrored.Filled.Send, "送信")
            }
        }
    }
}
```

---

## Scaffoldとの組み合わせ

```kotlin
@Composable
fun ScaffoldWithInsets() {
    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("App") },
                windowInsets = WindowInsets.statusBars  // ステータスバー考慮
            )
        },
        bottomBar = {
            NavigationBar(windowInsets = WindowInsets.navigationBars) {
                // ナビバー考慮
            }
        },
        contentWindowInsets = WindowInsets(0)  // コンテンツはScaffoldが管理
    ) { innerPadding ->
        Content(Modifier.padding(innerPadding))
    }
}
```

---

## ノッチ/カットアウト対応

```kotlin
@Composable
fun CutoutAwareScreen() {
    Box(
        Modifier
            .fillMaxSize()
            .windowInsetsPadding(WindowInsets.displayCutout)
    ) {
        // ノッチ/パンチホール領域を避けてコンテンツ配置
        Text("カットアウト安全領域内")
    }
}
```

---

## まとめ

| Inset | 対象 |
|-------|------|
| `statusBars` | ステータスバー |
| `navigationBars` | ナビゲーションバー |
| `ime` | キーボード |
| `systemBars` | status + nav |
| `displayCutout` | ノッチ |
| `safeDrawing` | 全安全領域 |

- `imePadding()`でキーボード表示時の自動調整
- `windowInsetsPadding()`で安全領域にコンテンツ配置
- `Scaffold`のwindowInsetsパラメータで統合管理
- Edge-to-Edgeと組み合わせて没入型UI

---

8種類のAndroidアプリテンプレート（Edge-to-Edge対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Edge-to-Edge](https://zenn.dev/myougatheaxo/articles/android-compose-edge-to-edge-2026)
- [キーボード/IME](https://zenn.dev/myougatheaxo/articles/android-compose-keyboard-ime-2026)
- [Scaffold](https://zenn.dev/myougatheaxo/articles/android-compose-scaffold-2026)
