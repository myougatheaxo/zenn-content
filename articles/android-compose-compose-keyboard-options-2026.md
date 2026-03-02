---
title: "KeyboardOptions完全ガイド — KeyboardType/ImeAction/AutoCorrect/Capitalization"
emoji: "⌨️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "textfield"]
published: true
---

## この記事で学べること

**KeyboardOptions**（KeyboardType、ImeAction、AutoCorrect、Capitalization）を解説します。

---

## KeyboardType

```kotlin
@Composable
fun KeyboardTypeExamples() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(8.dp)) {
        var email by remember { mutableStateOf("") }
        OutlinedTextField(
            value = email, onValueChange = { email = it },
            label = { Text("メール") },
            keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Email),
            modifier = Modifier.fillMaxWidth()
        )

        var phone by remember { mutableStateOf("") }
        OutlinedTextField(
            value = phone, onValueChange = { phone = it },
            label = { Text("電話番号") },
            keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Phone),
            modifier = Modifier.fillMaxWidth()
        )

        var amount by remember { mutableStateOf("") }
        OutlinedTextField(
            value = amount, onValueChange = { amount = it },
            label = { Text("金額") },
            keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Decimal),
            modifier = Modifier.fillMaxWidth()
        )

        var password by remember { mutableStateOf("") }
        OutlinedTextField(
            value = password, onValueChange = { password = it },
            label = { Text("パスワード") },
            keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Password),
            visualTransformation = PasswordVisualTransformation(),
            modifier = Modifier.fillMaxWidth()
        )

        var url by remember { mutableStateOf("") }
        OutlinedTextField(
            value = url, onValueChange = { url = it },
            label = { Text("URL") },
            keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Uri),
            modifier = Modifier.fillMaxWidth()
        )
    }
}
```

---

## ImeAction連携

```kotlin
@Composable
fun ImeActionForm() {
    val (nameFocus, emailFocus) = remember { FocusRequester.createRefs() }
    var name by remember { mutableStateOf("") }
    var email by remember { mutableStateOf("") }

    Column(Modifier.padding(16.dp)) {
        OutlinedTextField(
            value = name, onValueChange = { name = it },
            label = { Text("名前") },
            keyboardOptions = KeyboardOptions(
                imeAction = ImeAction.Next,
                capitalization = KeyboardCapitalization.Words
            ),
            keyboardActions = KeyboardActions(onNext = { emailFocus.requestFocus() }),
            modifier = Modifier.fillMaxWidth().focusRequester(nameFocus)
        )
        Spacer(Modifier.height(8.dp))
        OutlinedTextField(
            value = email, onValueChange = { email = it },
            label = { Text("メール") },
            keyboardOptions = KeyboardOptions(
                keyboardType = KeyboardType.Email,
                imeAction = ImeAction.Done
            ),
            keyboardActions = KeyboardActions(onDone = { /* submit */ }),
            modifier = Modifier.fillMaxWidth().focusRequester(emailFocus)
        )
    }
}
```

---

## まとめ

| オプション | 用途 |
|----------|------|
| `KeyboardType` | キーボード種類 |
| `ImeAction` | IMEボタン動作 |
| `KeyboardCapitalization` | 大文字変換 |
| `autoCorrect` | 自動修正 |

- `KeyboardType`でEmail/Phone/Decimal等を指定
- `ImeAction.Next`でフォーカス移動
- `KeyboardCapitalization.Words`で単語先頭を大文字
- `KeyboardActions`でIMEアクションのハンドリング

---

8種類のAndroidアプリテンプレート（フォーム対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [TextField/バリデーション](https://zenn.dev/myougatheaxo/articles/android-compose-textfield-validation-2026)
- [フォーカス管理](https://zenn.dev/myougatheaxo/articles/android-compose-compose-focus-2026)
- [キーボード/IME](https://zenn.dev/myougatheaxo/articles/android-compose-compose-keyboard-ime-2026)
