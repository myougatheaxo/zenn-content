---
title: "Compose KeyboardActions完全ガイド — IMEアクション/Done/Next/Search処理"
emoji: "⌨️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**Compose KeyboardActions**（onDone、onNext、onSearch、onGo、IMEアクション処理）を解説します。

---

## 基本KeyboardActions

```kotlin
@Composable
fun KeyboardActionsDemo() {
    val focusManager = LocalFocusManager.current
    var text by remember { mutableStateOf("") }

    OutlinedTextField(
        value = text,
        onValueChange = { text = it },
        label = { Text("入力") },
        keyboardOptions = KeyboardOptions(imeAction = ImeAction.Done),
        keyboardActions = KeyboardActions(
            onDone = {
                focusManager.clearFocus()
                // 送信処理
            }
        ),
        singleLine = true,
        modifier = Modifier.fillMaxWidth()
    )
}
```

---

## フォーム連携

```kotlin
@Composable
fun LoginForm(onLogin: (String, String) -> Unit) {
    val (emailRef, passwordRef) = remember { FocusRequester.createRefs() }
    val focusManager = LocalFocusManager.current
    var email by remember { mutableStateOf("") }
    var password by remember { mutableStateOf("") }

    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        OutlinedTextField(
            value = email,
            onValueChange = { email = it },
            label = { Text("メール") },
            modifier = Modifier.fillMaxWidth().focusRequester(emailRef),
            keyboardOptions = KeyboardOptions(
                keyboardType = KeyboardType.Email,
                imeAction = ImeAction.Next
            ),
            keyboardActions = KeyboardActions(onNext = { passwordRef.requestFocus() }),
            singleLine = true
        )

        OutlinedTextField(
            value = password,
            onValueChange = { password = it },
            label = { Text("パスワード") },
            modifier = Modifier.fillMaxWidth().focusRequester(passwordRef),
            visualTransformation = PasswordVisualTransformation(),
            keyboardOptions = KeyboardOptions(
                keyboardType = KeyboardType.Password,
                imeAction = ImeAction.Go
            ),
            keyboardActions = KeyboardActions(
                onGo = {
                    focusManager.clearFocus()
                    onLogin(email, password)
                }
            ),
            singleLine = true
        )

        Button(onClick = { onLogin(email, password) }, Modifier.fillMaxWidth()) {
            Text("ログイン")
        }
    }
}
```

---

## 検索フィールド

```kotlin
@Composable
fun SearchField(onSearch: (String) -> Unit) {
    var query by remember { mutableStateOf("") }
    val focusManager = LocalFocusManager.current

    OutlinedTextField(
        value = query,
        onValueChange = { query = it },
        placeholder = { Text("キーワード検索") },
        leadingIcon = { Icon(Icons.Default.Search, null) },
        keyboardOptions = KeyboardOptions(imeAction = ImeAction.Search),
        keyboardActions = KeyboardActions(
            onSearch = {
                focusManager.clearFocus()
                onSearch(query)
            }
        ),
        singleLine = true,
        modifier = Modifier.fillMaxWidth()
    )
}
```

---

## まとめ

| IMEアクション | 用途 |
|-------------|------|
| `ImeAction.Done` | 入力完了 |
| `ImeAction.Next` | 次フィールドへ |
| `ImeAction.Search` | 検索実行 |
| `ImeAction.Go` | 送信/実行 |

- `KeyboardActions`でIMEボタンタップ時の処理を定義
- `KeyboardOptions`の`imeAction`と対応させる
- `FocusRequester`と組み合わせてフィールド間移動
- `clearFocus()`でキーボードを閉じる

---

8種類のAndroidアプリテンプレート（フォーム対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose FocusRequester](https://zenn.dev/myougatheaxo/articles/android-compose-compose-focus-requester-2026)
- [Compose KeyboardOptions](https://zenn.dev/myougatheaxo/articles/android-compose-compose-keyboard-options-2026)
- [Compose InputFilter](https://zenn.dev/myougatheaxo/articles/android-compose-compose-input-filter-2026)
