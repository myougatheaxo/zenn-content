---
title: "Edge-to-Edge対応ガイド — Compose版（Android 15対応）"
emoji: "🖥️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "systemui"]
published: true
---

## この記事で学べること

**Edge-to-Edge**（システムバー背後へのコンテンツ拡張）をComposeで実装する方法を解説します。

---

## enableEdgeToEdge

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        enableEdgeToEdge() // Edge-to-Edge有効化
        super.onCreate(savedInstanceState)

        setContent {
            MyAppTheme {
                MyApp()
            }
        }
    }
}
```

---

## WindowInsets対応

```kotlin
@Composable
fun EdgeToEdgeScreen() {
    Scaffold(
        modifier = Modifier.fillMaxSize(),
        topBar = {
            TopAppBar(
                title = { Text("Edge-to-Edge") },
                // ステータスバー分のpadding自動適用
            )
        },
        contentWindowInsets = WindowInsets(0) // Scaffoldのデフォルトinsetを無効化
    ) { innerPadding ->
        LazyColumn(
            modifier = Modifier.fillMaxSize(),
            contentPadding = PaddingValues(
                top = innerPadding.calculateTopPadding(),
                bottom = innerPadding.calculateBottomPadding()
            )
        ) {
            items(50) { index ->
                ListItem(headlineContent = { Text("Item $index") })
            }
        }
    }
}
```

---

## Modifier.windowInsetsPadding

```kotlin
@Composable
fun InsetsPaddingExample() {
    Column(
        modifier = Modifier
            .fillMaxSize()
            .windowInsetsPadding(WindowInsets.systemBars)
    ) {
        Text("ステータスバーとナビバーの内側に配置")
    }
}

// 個別指定
@Composable
fun IndividualInsets() {
    Column(Modifier.fillMaxSize()) {
        // ステータスバー分だけパディング
        Box(Modifier.windowInsetsPadding(WindowInsets.statusBars)) {
            Text("ステータスバー下")
        }

        Spacer(Modifier.weight(1f))

        // ナビゲーションバー分だけパディング
        Box(Modifier.windowInsetsPadding(WindowInsets.navigationBars)) {
            Text("ナビバー上")
        }
    }
}
```

---

## IME（キーボード）対応

```kotlin
@Composable
fun ChatScreen() {
    Column(
        modifier = Modifier
            .fillMaxSize()
            .imePadding() // キーボード表示時に自動調整
    ) {
        LazyColumn(
            modifier = Modifier.weight(1f),
            reverseLayout = true
        ) {
            items(messages) { msg -> MessageItem(msg) }
        }

        // 入力欄はキーボードの上に配置
        Row(
            modifier = Modifier
                .fillMaxWidth()
                .navigationBarsPadding() // ナビバーの上に
                .padding(8.dp)
        ) {
            OutlinedTextField(
                value = text,
                onValueChange = { text = it },
                modifier = Modifier.weight(1f)
            )
            IconButton(onClick = { /* 送信 */ }) {
                Icon(Icons.Default.Send, "送信")
            }
        }
    }
}
```

---

## まとめ

- `enableEdgeToEdge()`でシステムバー背後まで描画
- `WindowInsets.systemBars`でステータスバー+ナビバーのインセット
- `Modifier.windowInsetsPadding()`で個別にパディング適用
- `Modifier.imePadding()`でキーボード表示時の自動調整
- `Scaffold`は自動でTopAppBarにステータスバー分を加算
- Android 15ではEdge-to-Edgeがデフォルト必須

---

8種類のAndroidアプリテンプレート（Edge-to-Edge対応済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [BottomNavigationガイド](https://zenn.dev/myougatheaxo/articles/android-compose-bottom-nav-scaffold-2026)
- [テーマ切り替え](https://zenn.dev/myougatheaxo/articles/android-compose-theme-switcher-2026)
- [Modifierチートシート](https://zenn.dev/myougatheaxo/articles/android-compose-modifier-cheatsheet-2026)
