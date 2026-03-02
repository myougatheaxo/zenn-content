---
title: "TextField詳細ガイド — バリデーション/マスク/フォーカス制御"
emoji: "✏️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "textfield"]
published: true
---

## この記事で学べること

ComposeのTextField詳細（バリデーション、入力マスク、フォーカス制御、VisualTransformation）を解説します。

---

## OutlinedTextField + バリデーション

```kotlin
@Composable
fun ValidatedEmailField() {
    var email by remember { mutableStateOf("") }
    var isError by remember { mutableStateOf(false) }

    OutlinedTextField(
        value = email,
        onValueChange = {
            email = it
            isError = it.isNotEmpty() && !Patterns.EMAIL_ADDRESS.matcher(it).matches()
        },
        label = { Text("メールアドレス") },
        isError = isError,
        supportingText = {
            if (isError) {
                Text("有効なメールアドレスを入力してください", color = MaterialTheme.colorScheme.error)
            }
        },
        leadingIcon = { Icon(Icons.Default.Email, null) },
        trailingIcon = {
            if (isError) Icon(Icons.Default.Error, null, tint = MaterialTheme.colorScheme.error)
        },
        keyboardOptions = KeyboardOptions(
            keyboardType = KeyboardType.Email,
            imeAction = ImeAction.Next
        ),
        singleLine = true,
        modifier = Modifier.fillMaxWidth()
    )
}
```

---

## パスワードフィールド

```kotlin
@Composable
fun PasswordField() {
    var password by remember { mutableStateOf("") }
    var visible by remember { mutableStateOf(false) }

    OutlinedTextField(
        value = password,
        onValueChange = { password = it },
        label = { Text("パスワード") },
        visualTransformation = if (visible) VisualTransformation.None
            else PasswordVisualTransformation(),
        trailingIcon = {
            IconButton(onClick = { visible = !visible }) {
                Icon(
                    if (visible) Icons.Default.VisibilityOff else Icons.Default.Visibility,
                    contentDescription = if (visible) "非表示" else "表示"
                )
            }
        },
        keyboardOptions = KeyboardOptions(
            keyboardType = KeyboardType.Password,
            imeAction = ImeAction.Done
        ),
        singleLine = true,
        modifier = Modifier.fillMaxWidth()
    )
}
```

---

## フォーカス制御

```kotlin
@Composable
fun FocusControlForm() {
    val focusManager = LocalFocusManager.current
    val (emailFocus, passwordFocus, confirmFocus) = remember { FocusRequester.createRefs() }

    var email by remember { mutableStateOf("") }
    var password by remember { mutableStateOf("") }
    var confirm by remember { mutableStateOf("") }

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
            keyboardOptions = KeyboardOptions(imeAction = ImeAction.Next),
            keyboardActions = KeyboardActions(onNext = { confirmFocus.requestFocus() }),
            modifier = Modifier.fillMaxWidth().focusRequester(passwordFocus)
        )

        Spacer(Modifier.height(8.dp))

        OutlinedTextField(
            value = confirm,
            onValueChange = { confirm = it },
            label = { Text("パスワード確認") },
            visualTransformation = PasswordVisualTransformation(),
            keyboardOptions = KeyboardOptions(imeAction = ImeAction.Done),
            keyboardActions = KeyboardActions(onDone = { focusManager.clearFocus() }),
            modifier = Modifier.fillMaxWidth().focusRequester(confirmFocus)
        )
    }

    // 初回フォーカス
    LaunchedEffect(Unit) {
        emailFocus.requestFocus()
    }
}
```

---

## VisualTransformation（電話番号マスク）

```kotlin
class PhoneVisualTransformation : VisualTransformation {
    override fun filter(text: AnnotatedString): TransformedText {
        val digits = text.text.filter { it.isDigit() }
        val formatted = buildString {
            digits.forEachIndexed { i, c ->
                if (i == 3 || i == 7) append('-')
                append(c)
            }
        }

        val offsetMapping = object : OffsetMapping {
            override fun originalToTransformed(offset: Int): Int {
                return when {
                    offset <= 3 -> offset
                    offset <= 7 -> offset + 1
                    else -> offset + 2
                }.coerceAtMost(formatted.length)
            }

            override fun transformedToOriginal(offset: Int): Int {
                return when {
                    offset <= 3 -> offset
                    offset <= 8 -> offset - 1
                    else -> offset - 2
                }.coerceAtMost(digits.length)
            }
        }

        return TransformedText(AnnotatedString(formatted), offsetMapping)
    }
}

// 使用
OutlinedTextField(
    value = phone,
    onValueChange = { phone = it.filter { c -> c.isDigit() }.take(11) },
    label = { Text("電話番号") },
    visualTransformation = PhoneVisualTransformation(),
    keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Phone)
)
```

---

## 文字数制限 + カウンター

```kotlin
@Composable
fun LimitedTextField(maxLength: Int = 100) {
    var text by remember { mutableStateOf("") }

    OutlinedTextField(
        value = text,
        onValueChange = { if (it.length <= maxLength) text = it },
        label = { Text("コメント") },
        supportingText = {
            Text(
                "${text.length} / $maxLength",
                modifier = Modifier.fillMaxWidth(),
                textAlign = TextAlign.End
            )
        },
        minLines = 3,
        maxLines = 5,
        modifier = Modifier.fillMaxWidth()
    )
}
```

---

## まとめ

- `OutlinedTextField`の`isError` + `supportingText`でバリデーション表示
- `PasswordVisualTransformation`でパスワードマスク
- `FocusRequester` + `KeyboardActions`でフォーカス遷移
- `VisualTransformation`で電話番号・クレカ番号のマスク
- `onValueChange`で文字数制限・入力フィルタ
- `ImeAction`でキーボードのアクションボタン制御

---

8種類のAndroidアプリテンプレート（フォーム実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [フォームウィザード](https://zenn.dev/myougatheaxo/articles/android-compose-form-wizard-2026)
- [パスワード強度表示](https://zenn.dev/myougatheaxo/articles/android-compose-password-strength-2026)
- [キーボード/IME](https://zenn.dev/myougatheaxo/articles/android-compose-keyboard-ime-2026)
