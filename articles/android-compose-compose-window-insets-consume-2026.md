---
title: "WindowInsets消費完全ガイド — consumeWindowInsets/imePadding/ステータスバー制御"
emoji: "📐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "windowinsets"]
published: true
---

## この記事で学べること

**WindowInsets消費**（consumeWindowInsets、imePadding、ステータスバー・ナビゲーションバー制御）を解説します。

---

## WindowInsets基本

```kotlin
@Composable
fun WindowInsetsExample() {
    Scaffold(
        modifier = Modifier.fillMaxSize(),
        contentWindowInsets = WindowInsets(0)  // Scaffoldのデフォルトinsets無効化
    ) { innerPadding ->
        Column(
            Modifier
                .padding(innerPadding)
                .consumeWindowInsets(innerPadding)
                .systemBarsPadding()
        ) {
            Text("システムバー対応コンテンツ")
        }
    }
}
```

---

## imePadding

```kotlin
@Composable
fun ImePaddingForm() {
    val scrollState = rememberScrollState()

    Column(
        Modifier
            .fillMaxSize()
            .statusBarsPadding()
            .navigationBarsPadding()
            .imePadding()
            .verticalScroll(scrollState)
            .padding(16.dp)
    ) {
        var name by remember { mutableStateOf("") }
        var email by remember { mutableStateOf("") }
        var message by remember { mutableStateOf("") }

        OutlinedTextField(
            value = name, onValueChange = { name = it },
            label = { Text("名前") },
            modifier = Modifier.fillMaxWidth()
        )
        Spacer(Modifier.height(8.dp))
        OutlinedTextField(
            value = email, onValueChange = { email = it },
            label = { Text("メール") },
            keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Email),
            modifier = Modifier.fillMaxWidth()
        )
        Spacer(Modifier.height(8.dp))
        OutlinedTextField(
            value = message, onValueChange = { message = it },
            label = { Text("メッセージ") },
            minLines = 4,
            modifier = Modifier.fillMaxWidth()
        )
    }
}
```

---

## カスタムInsets消費

```kotlin
@Composable
fun CustomInsetsConsumption() {
    Column(Modifier.fillMaxSize()) {
        // ヘッダー：ステータスバー分のパディング消費
        Surface(
            color = MaterialTheme.colorScheme.primary,
            modifier = Modifier
                .fillMaxWidth()
                .windowInsetsPadding(WindowInsets.statusBars)
        ) {
            Text(
                "ヘッダー",
                color = Color.White,
                modifier = Modifier.padding(16.dp),
                style = MaterialTheme.typography.titleLarge
            )
        }

        // コンテンツ：残りのInsetsのみ適用
        Box(
            Modifier
                .weight(1f)
                .consumeWindowInsets(WindowInsets.statusBars)
                .navigationBarsPadding()
                .padding(16.dp)
        ) {
            Text("メインコンテンツ")
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `systemBarsPadding()` | ステータス+ナビバー |
| `imePadding()` | キーボード表示対応 |
| `consumeWindowInsets` | Insets消費済みマーク |
| `statusBarsPadding()` | ステータスバーのみ |

- `consumeWindowInsets`で二重パディング防止
- `imePadding()`でキーボード表示時に自動調整
- `contentWindowInsets`でScaffoldのデフォルト制御
- Edge-to-Edgeと組み合わせて使用

---

8種類のAndroidアプリテンプレート（Edge-to-Edge対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Edge-to-Edge](https://zenn.dev/myougatheaxo/articles/android-compose-compose-edge-to-edge-2026)
- [キーボード/IME](https://zenn.dev/myougatheaxo/articles/android-compose-compose-keyboard-ime-2026)
- [Scaffold](https://zenn.dev/myougatheaxo/articles/android-compose-compose-scaffold-2026)
