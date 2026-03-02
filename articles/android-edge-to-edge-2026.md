---
title: "Edge-to-Edge表示完全ガイド — ステータスバー・ナビバーの透過対応"
emoji: "📱"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

Android 15からEdge-to-Edge（全画面表示）が**デフォルト**になりました。ステータスバーやナビゲーションバーの背後にコンテンツを描画する正しい方法を解説します。

---

## Edge-to-Edgeを有効化

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()  // これだけ

        setContent {
            MyAppTheme {
                MainScreen()
            }
        }
    }
}
```

`enableEdgeToEdge()`を呼ぶだけで、ステータスバーとナビゲーションバーが透過になります。

---

## WindowInsetsの処理

Edge-to-Edgeにすると、コンテンツがシステムバーの下に隠れます。`WindowInsets`でパディングを追加します。

```kotlin
@Composable
fun MainScreen() {
    Scaffold(
        modifier = Modifier.fillMaxSize(),
        topBar = {
            TopAppBar(
                title = { Text("My App") },
                modifier = Modifier.windowInsetsPadding(
                    WindowInsets.statusBars
                )
            )
        },
        bottomBar = {
            NavigationBar(
                modifier = Modifier.windowInsetsPadding(
                    WindowInsets.navigationBars
                )
            ) {
                // ナビゲーション項目
            }
        }
    ) { padding ->
        LazyColumn(
            modifier = Modifier.padding(padding)
        ) {
            // コンテンツ
        }
    }
}
```

---

## Scaffoldは自動対応

**Material3のScaffold**を使っていれば、`topBar`と`bottomBar`のInsets処理は**自動**です。

```kotlin
Scaffold(
    topBar = { TopAppBar(title = { Text("自動対応") }) },
    bottomBar = { NavigationBar { /* ... */ } }
) { innerPadding ->
    // innerPaddingにInsets含まれている
    Column(Modifier.padding(innerPadding)) {
        Text("Scaffoldなら自動")
    }
}
```

---

## 手動でInsetsを処理

Scaffoldを使わない場合は手動でInsets処理が必要です。

```kotlin
@Composable
fun CustomLayout() {
    val statusBarInsets = WindowInsets.statusBars
    val navBarInsets = WindowInsets.navigationBars

    Column(
        Modifier
            .fillMaxSize()
            .windowInsetsPadding(statusBarInsets)
    ) {
        Text("ステータスバーの下に表示")

        Spacer(Modifier.weight(1f))

        Button(
            onClick = { },
            modifier = Modifier
                .windowInsetsPadding(navBarInsets)
                .padding(16.dp)
        ) {
            Text("ナビバーの上に表示")
        }
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
            .imePadding()  // キーボードが出たらパディング追加
    ) {
        LazyColumn(
            Modifier.weight(1f),
            reverseLayout = true
        ) {
            // チャットメッセージ
        }

        OutlinedTextField(
            value = text,
            onValueChange = { text = it },
            modifier = Modifier
                .fillMaxWidth()
                .navigationBarsPadding()
        )
    }
}
```

`imePadding()`でソフトキーボード表示時にコンテンツが押し上げられます。

---

## よくある問題と解決

| 問題 | 原因 | 解決 |
|------|------|------|
| コンテンツがステータスバーに隠れる | Insetsパディング未設定 | `windowInsetsPadding(WindowInsets.statusBars)` |
| ボタンがナビバーに隠れる | 同上 | `navigationBarsPadding()` |
| キーボードでテキスト入力が隠れる | IME未対応 | `imePadding()` |
| 二重パディング | ScaffoldとInsetsの重複適用 | Scaffoldの`innerPadding`のみ使う |

---

## まとめ

- `enableEdgeToEdge()`で全画面表示を有効化
- **Scaffold**を使えばInsets処理は自動
- 手動の場合は`windowInsetsPadding()`で各Insetsに対応
- IMEには`imePadding()`
- Android 15からデフォルト有効なので対応必須

---

8種類のAndroidアプリテンプレート（Edge-to-Edge対応済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Composeレイアウト入門](https://zenn.dev/myougatheaxo/articles/compose-layout-basics-2026)
- [SplashScreen API完全ガイド](https://zenn.dev/myougatheaxo/articles/android-splash-screen-2026)
- [Material3テーマ完全ガイド](https://zenn.dev/myougatheaxo/articles/material3-theme-guide-2026)
