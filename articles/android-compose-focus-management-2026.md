---
title: "フォーカス管理ガイド — FocusRequester/FocusManager/IMEアクション"
emoji: "🎯"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "focus"]
published: true
---

## この記事で学べること

Composeでの**フォーカス管理**（FocusRequester、FocusManager、IMEアクション）を解説します。

---

## FocusRequester

```kotlin
@Composable
fun FocusRequestExample() {
    val focusRequester = remember { FocusRequester() }
    var text by remember { mutableStateOf("") }

    Column(Modifier.padding(16.dp)) {
        OutlinedTextField(
            value = text,
            onValueChange = { text = it },
            label = { Text("自動フォーカス") },
            modifier = Modifier
                .fillMaxWidth()
                .focusRequester(focusRequester)
        )

        // 画面表示時に自動フォーカス
        LaunchedEffect(Unit) {
            focusRequester.requestFocus()
        }
    }
}
```

---

## フィールド間のフォーカス移動

```kotlin
@Composable
fun LoginForm() {
    var email by remember { mutableStateOf("") }
    var password by remember { mutableStateOf("") }
    val passwordFocus = remember { FocusRequester() }
    val focusManager = LocalFocusManager.current

    Column(Modifier.padding(16.dp)) {
        OutlinedTextField(
            value = email,
            onValueChange = { email = it },
            label = { Text("メール") },
            keyboardOptions = KeyboardOptions(
                keyboardType = KeyboardType.Email,
                imeAction = ImeAction.Next
            ),
            keyboardActions = KeyboardActions(
                onNext = { passwordFocus.requestFocus() }
            ),
            modifier = Modifier.fillMaxWidth()
        )

        Spacer(Modifier.height(8.dp))

        OutlinedTextField(
            value = password,
            onValueChange = { password = it },
            label = { Text("パスワード") },
            visualTransformation = PasswordVisualTransformation(),
            keyboardOptions = KeyboardOptions(
                keyboardType = KeyboardType.Password,
                imeAction = ImeAction.Done
            ),
            keyboardActions = KeyboardActions(
                onDone = {
                    focusManager.clearFocus()
                    // ログイン処理
                }
            ),
            modifier = Modifier
                .fillMaxWidth()
                .focusRequester(passwordFocus)
        )

        Spacer(Modifier.height(16.dp))

        Button(
            onClick = { focusManager.clearFocus() },
            modifier = Modifier.fillMaxWidth()
        ) {
            Text("ログイン")
        }
    }
}
```

---

## 複数フィールドのフォーカス移動

```kotlin
@Composable
fun MultiFieldForm() {
    val fields = listOf("名前", "メール", "電話", "住所")
    val values = remember { mutableStateListOf("", "", "", "") }
    val focusRequesters = remember { fields.map { FocusRequester() } }
    val focusManager = LocalFocusManager.current

    Column(Modifier.padding(16.dp)) {
        fields.forEachIndexed { index, label ->
            OutlinedTextField(
                value = values[index],
                onValueChange = { values[index] = it },
                label = { Text(label) },
                keyboardOptions = KeyboardOptions(
                    imeAction = if (index < fields.lastIndex) ImeAction.Next else ImeAction.Done
                ),
                keyboardActions = KeyboardActions(
                    onNext = {
                        if (index < fields.lastIndex) {
                            focusRequesters[index + 1].requestFocus()
                        }
                    },
                    onDone = { focusManager.clearFocus() }
                ),
                modifier = Modifier
                    .fillMaxWidth()
                    .focusRequester(focusRequesters[index])
            )
            Spacer(Modifier.height(8.dp))
        }
    }
}
```

---

## フォーカス状態の検出

```kotlin
@Composable
fun FocusAwareField() {
    var text by remember { mutableStateOf("") }
    var isFocused by remember { mutableStateOf(false) }

    OutlinedTextField(
        value = text,
        onValueChange = { text = it },
        label = { Text("入力フィールド") },
        modifier = Modifier
            .fillMaxWidth()
            .onFocusChanged { focusState ->
                isFocused = focusState.isFocused
            }
            .border(
                width = if (isFocused) 2.dp else 0.dp,
                color = if (isFocused) MaterialTheme.colorScheme.primary else Color.Transparent,
                shape = RoundedCornerShape(8.dp)
            )
    )

    if (isFocused) {
        Text(
            "入力中...",
            style = MaterialTheme.typography.bodySmall,
            color = MaterialTheme.colorScheme.primary
        )
    }
}
```

---

## 外部タップでフォーカス解除

```kotlin
@Composable
fun DismissKeyboardOnTap() {
    val focusManager = LocalFocusManager.current

    Box(
        Modifier
            .fillMaxSize()
            .pointerInput(Unit) {
                detectTapGestures {
                    focusManager.clearFocus()
                }
            }
    ) {
        Column(Modifier.padding(16.dp)) {
            var text by remember { mutableStateOf("") }
            OutlinedTextField(
                value = text,
                onValueChange = { text = it },
                label = { Text("タップでキーボード閉じる") },
                modifier = Modifier.fillMaxWidth()
            )
        }
    }
}
```

---

## まとめ

- `FocusRequester`で特定フィールドにフォーカスを設定
- `ImeAction.Next`/`Done`でキーボードアクション
- `KeyboardActions`でIMEアクション時の処理
- `LocalFocusManager.current.clearFocus()`でフォーカス解除
- `onFocusChanged`でフォーカス状態を監視
- `detectTapGestures`で外部タップ時にキーボード閉じる

---

8種類のAndroidアプリテンプレート（フォーム入力設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [入力フォーム完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-forms-input-2026)
- [キーボード/IMEガイド](https://zenn.dev/myougatheaxo/articles/android-compose-keyboard-ime-2026)
- [パスワード強度チェッカー](https://zenn.dev/myougatheaxo/articles/android-compose-password-strength-2026)
