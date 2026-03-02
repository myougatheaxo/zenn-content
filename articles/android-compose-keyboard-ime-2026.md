---
title: "キーボード・IME対応ガイド — Composeでソフトキーボードを制御する"
emoji: "⌨️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

Composeでの**ソフトキーボード制御**・IME対応・フォーカス管理の実装方法を解説します。

---

## imePadding（キーボードで隠れない）

```kotlin
@Composable
fun ChatScreen() {
    Scaffold { padding ->
        Column(
            Modifier
                .padding(padding)
                .fillMaxSize()
                .imePadding()  // キーボード分のパディング
        ) {
            LazyColumn(
                modifier = Modifier.weight(1f),
                reverseLayout = true
            ) {
                items(messages) { message ->
                    MessageItem(message)
                }
            }

            // 入力欄（キーボードの上に表示される）
            Row(Modifier.padding(8.dp)) {
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
}
```

---

## キーボードの表示/非表示

```kotlin
@Composable
fun KeyboardControl() {
    val keyboardController = LocalSoftwareKeyboardController.current
    val focusManager = LocalFocusManager.current

    Column {
        OutlinedTextField(
            value = text,
            onValueChange = { text = it },
            keyboardActions = KeyboardActions(
                onDone = {
                    keyboardController?.hide()
                    focusManager.clearFocus()
                }
            ),
            keyboardOptions = KeyboardOptions(imeAction = ImeAction.Done)
        )

        Button(onClick = {
            keyboardController?.hide()
            focusManager.clearFocus()
        }) {
            Text("キーボードを閉じる")
        }
    }
}
```

---

## KeyboardOptions（キーボードタイプ）

```kotlin
// メールアドレス
OutlinedTextField(
    keyboardOptions = KeyboardOptions(
        keyboardType = KeyboardType.Email,
        imeAction = ImeAction.Next
    )
)

// 電話番号
OutlinedTextField(
    keyboardOptions = KeyboardOptions(
        keyboardType = KeyboardType.Phone,
        imeAction = ImeAction.Done
    )
)

// 数字（金額入力）
OutlinedTextField(
    keyboardOptions = KeyboardOptions(
        keyboardType = KeyboardType.Decimal,
        imeAction = ImeAction.Done
    )
)

// パスワード
OutlinedTextField(
    keyboardOptions = KeyboardOptions(
        keyboardType = KeyboardType.Password,
        imeAction = ImeAction.Go
    ),
    visualTransformation = PasswordVisualTransformation()
)
```

---

## フォーカス移動

```kotlin
@Composable
fun FormWithFocus() {
    val focusManager = LocalFocusManager.current
    var name by remember { mutableStateOf("") }
    var email by remember { mutableStateOf("") }
    var password by remember { mutableStateOf("") }

    Column(Modifier.padding(16.dp)) {
        OutlinedTextField(
            value = name,
            onValueChange = { name = it },
            label = { Text("名前") },
            keyboardOptions = KeyboardOptions(imeAction = ImeAction.Next),
            keyboardActions = KeyboardActions(
                onNext = { focusManager.moveFocus(FocusDirection.Down) }
            ),
            modifier = Modifier.fillMaxWidth()
        )

        OutlinedTextField(
            value = email,
            onValueChange = { email = it },
            label = { Text("メール") },
            keyboardOptions = KeyboardOptions(
                keyboardType = KeyboardType.Email,
                imeAction = ImeAction.Next
            ),
            keyboardActions = KeyboardActions(
                onNext = { focusManager.moveFocus(FocusDirection.Down) }
            ),
            modifier = Modifier.fillMaxWidth()
        )

        OutlinedTextField(
            value = password,
            onValueChange = { password = it },
            label = { Text("パスワード") },
            keyboardOptions = KeyboardOptions(
                keyboardType = KeyboardType.Password,
                imeAction = ImeAction.Done
            ),
            keyboardActions = KeyboardActions(
                onDone = {
                    focusManager.clearFocus()
                    // ログイン実行
                }
            ),
            visualTransformation = PasswordVisualTransformation(),
            modifier = Modifier.fillMaxWidth()
        )
    }
}
```

---

## FocusRequester（プログラムでフォーカス）

```kotlin
@Composable
fun AutoFocusTextField() {
    val focusRequester = remember { FocusRequester() }
    var text by remember { mutableStateOf("") }

    LaunchedEffect(Unit) {
        focusRequester.requestFocus()
    }

    OutlinedTextField(
        value = text,
        onValueChange = { text = it },
        modifier = Modifier
            .fillMaxWidth()
            .focusRequester(focusRequester)
    )
}
```

---

## まとめ

- `imePadding()`でキーボードに合わせたレイアウト調整
- `LocalSoftwareKeyboardController`でキーボード制御
- `KeyboardOptions`でキーボードタイプ指定
- `KeyboardActions` + `FocusManager`でフォーカス移動
- `FocusRequester`でプログラム的にフォーカス設定

---

8種類のAndroidアプリテンプレート（入力フォーム設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [入力フォーム完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-forms-input-2026)
- [Edge-to-Edge完全ガイド](https://zenn.dev/myougatheaxo/articles/android-edge-to-edge-2026)
- [BottomSheet完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-bottom-sheet-2026)
