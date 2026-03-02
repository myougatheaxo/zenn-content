---
title: "フォーカス管理完全ガイド — FocusRequester/FocusManager/Tab順序/フォーカストラップ"
emoji: "🎯"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**フォーカス管理**（FocusRequester、FocusManager、Tab順序、フォーカストラップ）を解説します。

---

## FocusRequester

```kotlin
@Composable
fun FocusExample() {
    val (emailFocus, passwordFocus, submitFocus) = remember { FocusRequester.createRefs() }
    var email by remember { mutableStateOf("") }
    var password by remember { mutableStateOf("") }

    LaunchedEffect(Unit) { emailFocus.requestFocus() }

    Column(Modifier.padding(16.dp)) {
        OutlinedTextField(
            value = email,
            onValueChange = { email = it },
            label = { Text("メール") },
            keyboardOptions = KeyboardOptions(imeAction = ImeAction.Next),
            keyboardActions = KeyboardActions(onNext = { passwordFocus.requestFocus() }),
            modifier = Modifier.fillMaxWidth().focusRequester(emailFocus)
        )

        Spacer(Modifier.height(8.dp))

        OutlinedTextField(
            value = password,
            onValueChange = { password = it },
            label = { Text("パスワード") },
            visualTransformation = PasswordVisualTransformation(),
            keyboardOptions = KeyboardOptions(imeAction = ImeAction.Done),
            keyboardActions = KeyboardActions(onDone = { submitFocus.requestFocus() }),
            modifier = Modifier.fillMaxWidth().focusRequester(passwordFocus)
        )

        Spacer(Modifier.height(16.dp))

        Button(
            onClick = {},
            modifier = Modifier.fillMaxWidth().focusRequester(submitFocus)
        ) { Text("ログイン") }
    }
}
```

---

## フォーカス状態監視

```kotlin
@Composable
fun FocusAwareTextField() {
    var text by remember { mutableStateOf("") }
    var isFocused by remember { mutableStateOf(false) }

    OutlinedTextField(
        value = text,
        onValueChange = { text = it },
        label = { Text("入力") },
        modifier = Modifier
            .fillMaxWidth()
            .onFocusChanged { focusState ->
                isFocused = focusState.isFocused
            }
            .border(
                width = if (isFocused) 2.dp else 1.dp,
                color = if (isFocused) MaterialTheme.colorScheme.primary else Color.Gray,
                shape = RoundedCornerShape(8.dp)
            )
    )

    if (isFocused) {
        Text("入力中...", color = MaterialTheme.colorScheme.primary, fontSize = 12.sp)
    }
}
```

---

## フォーカスクリア

```kotlin
@Composable
fun DismissKeyboardOnTap() {
    val focusManager = LocalFocusManager.current
    var text by remember { mutableStateOf("") }

    Box(
        Modifier
            .fillMaxSize()
            .clickable(
                interactionSource = remember { MutableInteractionSource() },
                indication = null
            ) {
                focusManager.clearFocus()
            }
    ) {
        Column(Modifier.padding(16.dp)) {
            OutlinedTextField(
                value = text,
                onValueChange = { text = it },
                label = { Text("タップ外でキーボード閉じる") },
                modifier = Modifier.fillMaxWidth()
            )
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `FocusRequester` | フォーカス要求 |
| `onFocusChanged` | 状態監視 |
| `FocusManager` | フォーカス解除 |
| `focusProperties` | 移動順序 |

- `FocusRequester`でIME Nextアクション連携
- `onFocusChanged`でフォーカス状態に応じたUI変更
- `clearFocus()`でキーボードを閉じる
- `createRefs()`で複数フォーカス要求を管理

---

8種類のAndroidアプリテンプレート（フォーム対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [キーボード](https://zenn.dev/myougatheaxo/articles/android-compose-keyboard-shortcut-2026)
- [TextField/バリデーション](https://zenn.dev/myougatheaxo/articles/android-compose-textfield-validation-2026)
- [アクセシビリティ](https://zenn.dev/myougatheaxo/articles/android-compose-accessibility-2026)
