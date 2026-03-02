---
title: "キーボード/IME完全ガイド — WindowInsets/imeNestedScroll/キーボード検知"
emoji: "⌨️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**キーボード/IME**（WindowInsets.ime、imeNestedScroll、キーボード表示検知、レイアウト調整）を解説します。

---

## WindowInsets.ime

```kotlin
@Composable
fun ChatScreen() {
    var message by remember { mutableStateOf("") }
    val imeInsets = WindowInsets.ime

    Column(
        Modifier
            .fillMaxSize()
            .windowInsetsPadding(imeInsets)
    ) {
        // メッセージリスト
        LazyColumn(
            modifier = Modifier.weight(1f),
            reverseLayout = true
        ) {
            items(50) { index ->
                ListItem(headlineContent = { Text("メッセージ $index") })
            }
        }

        // 入力欄
        Row(
            Modifier
                .fillMaxWidth()
                .padding(8.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            OutlinedTextField(
                value = message,
                onValueChange = { message = it },
                placeholder = { Text("メッセージを入力") },
                modifier = Modifier.weight(1f)
            )
            IconButton(onClick = { message = "" }) {
                Icon(Icons.Default.Send, "送信")
            }
        }
    }
}
```

---

## キーボード表示検知

```kotlin
@Composable
fun KeyboardAwareLayout() {
    val imeVisible = WindowInsets.isImeVisible

    Column(Modifier.fillMaxSize()) {
        // キーボード表示時にヘッダーを隠す
        AnimatedVisibility(visible = !imeVisible) {
            Box(
                Modifier
                    .fillMaxWidth()
                    .height(200.dp)
                    .background(MaterialTheme.colorScheme.primaryContainer),
                contentAlignment = Alignment.Center
            ) {
                Text("ヘッダー画像", style = MaterialTheme.typography.headlineMedium)
            }
        }

        Column(Modifier.padding(16.dp)) {
            var email by remember { mutableStateOf("") }
            var password by remember { mutableStateOf("") }

            OutlinedTextField(
                value = email,
                onValueChange = { email = it },
                label = { Text("メール") },
                modifier = Modifier.fillMaxWidth()
            )
            Spacer(Modifier.height(8.dp))
            OutlinedTextField(
                value = password,
                onValueChange = { password = it },
                label = { Text("パスワード") },
                modifier = Modifier.fillMaxWidth()
            )
        }
    }
}
```

---

## imeNestedScroll

```kotlin
@Composable
fun ImeScrollExample() {
    var text by remember { mutableStateOf("") }

    Column(
        Modifier
            .fillMaxSize()
            .imePadding()
            .imeNestedScroll()
    ) {
        LazyColumn(Modifier.weight(1f)) {
            items(30) { index ->
                ListItem(headlineContent = { Text("Item $index") })
            }
        }

        OutlinedTextField(
            value = text,
            onValueChange = { text = it },
            modifier = Modifier
                .fillMaxWidth()
                .padding(8.dp),
            placeholder = { Text("入力...") }
        )
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `WindowInsets.ime` | IMEインセット |
| `imePadding()` | キーボード分パディング |
| `imeNestedScroll()` | スクロール連動 |
| `WindowInsets.isImeVisible` | キーボード表示判定 |

- `imePadding()`でキーボードに被らないレイアウト
- `isImeVisible`でキーボード表示/非表示を検知
- `imeNestedScroll()`でスクロールとキーボードを連動
- チャットUIは`reverseLayout = true`と組み合わせ

---

8種類のAndroidアプリテンプレート（キーボード対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [フォーカス管理](https://zenn.dev/myougatheaxo/articles/android-compose-compose-focus-2026)
- [TextField/バリデーション](https://zenn.dev/myougatheaxo/articles/android-compose-textfield-validation-2026)
- [キーボードショートカット](https://zenn.dev/myougatheaxo/articles/android-compose-keyboard-shortcut-2026)
