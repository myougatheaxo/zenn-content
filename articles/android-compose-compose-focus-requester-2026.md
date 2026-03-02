---
title: "Compose FocusRequester完全ガイド — フォーカス制御/Tab移動/自動フォーカス"
emoji: "🎯"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**Compose FocusRequester**（フォーカス制御、フォーカス移動、自動フォーカス、カスタムフォーカス順序）を解説します。

---

## 基本FocusRequester

```kotlin
@Composable
fun FocusDemo() {
    val (nameRef, emailRef, passwordRef) = remember { FocusRequester.createRefs() }

    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        var name by remember { mutableStateOf("") }
        var email by remember { mutableStateOf("") }
        var password by remember { mutableStateOf("") }

        OutlinedTextField(
            value = name,
            onValueChange = { name = it },
            label = { Text("名前") },
            modifier = Modifier.fillMaxWidth().focusRequester(nameRef),
            keyboardOptions = KeyboardOptions(imeAction = ImeAction.Next),
            keyboardActions = KeyboardActions(onNext = { emailRef.requestFocus() })
        )
        OutlinedTextField(
            value = email,
            onValueChange = { email = it },
            label = { Text("メール") },
            modifier = Modifier.fillMaxWidth().focusRequester(emailRef),
            keyboardOptions = KeyboardOptions(imeAction = ImeAction.Next),
            keyboardActions = KeyboardActions(onNext = { passwordRef.requestFocus() })
        )
        OutlinedTextField(
            value = password,
            onValueChange = { password = it },
            label = { Text("パスワード") },
            modifier = Modifier.fillMaxWidth().focusRequester(passwordRef),
            keyboardOptions = KeyboardOptions(imeAction = ImeAction.Done),
            keyboardActions = KeyboardActions(onDone = { /* submit */ })
        )
    }

    // 初期フォーカス
    LaunchedEffect(Unit) {
        nameRef.requestFocus()
    }
}
```

---

## フォーカス状態の監視

```kotlin
@Composable
fun FocusStateDemo() {
    var text by remember { mutableStateOf("") }
    var isFocused by remember { mutableStateOf(false) }

    OutlinedTextField(
        value = text,
        onValueChange = { text = it },
        label = { Text("入力欄") },
        modifier = Modifier
            .fillMaxWidth()
            .onFocusChanged { isFocused = it.isFocused }
            .border(
                width = if (isFocused) 2.dp else 1.dp,
                color = if (isFocused) Color.Blue else Color.Gray,
                shape = RoundedCornerShape(8.dp)
            )
    )

    if (isFocused) {
        Text("入力中...", color = Color.Blue, style = MaterialTheme.typography.bodySmall)
    }
}
```

---

## 検索バー自動フォーカス

```kotlin
@Composable
fun SearchScreen() {
    val focusRequester = remember { FocusRequester() }
    val focusManager = LocalFocusManager.current
    var query by remember { mutableStateOf("") }

    Column {
        OutlinedTextField(
            value = query,
            onValueChange = { query = it },
            placeholder = { Text("検索...") },
            modifier = Modifier.fillMaxWidth().focusRequester(focusRequester),
            leadingIcon = { Icon(Icons.Default.Search, null) },
            trailingIcon = {
                if (query.isNotEmpty()) {
                    IconButton(onClick = { query = ""; focusRequester.requestFocus() }) {
                        Icon(Icons.Default.Clear, "クリア")
                    }
                }
            },
            keyboardActions = KeyboardActions(onSearch = { focusManager.clearFocus() }),
            keyboardOptions = KeyboardOptions(imeAction = ImeAction.Search),
            singleLine = true
        )
    }

    LaunchedEffect(Unit) { focusRequester.requestFocus() }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `FocusRequester` | フォーカス制御 |
| `requestFocus()` | フォーカス要求 |
| `onFocusChanged` | フォーカス状態監視 |
| `clearFocus()` | フォーカス解除 |

- `FocusRequester.createRefs()`で複数フィールドのフォーカス管理
- `KeyboardActions`と組み合わせてIMEアクションでフォーカス移動
- `LaunchedEffect`で画面表示時に自動フォーカス
- `LocalFocusManager`でフォーカスクリア

---

8種類のAndroidアプリテンプレート（フォーム対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose KeyboardActions](https://zenn.dev/myougatheaxo/articles/android-compose-compose-keyboard-actions-2026)
- [Compose KeyboardOptions](https://zenn.dev/myougatheaxo/articles/android-compose-compose-keyboard-options-2026)
- [Compose TextFieldState](https://zenn.dev/myougatheaxo/articles/android-compose-compose-text-field-state-2026)
