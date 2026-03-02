---
title: "Compose AutoFill完全ガイド — オートフィル対応/フォーム自動入力/セマンティクス"
emoji: "📋"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "autofill"]
published: true
---

## この記事で学べること

**Compose AutoFill**（オートフィル対応、セマンティクスヒント、パスワードマネージャー連携）を解説します。

---

## 基本AutoFill

```kotlin
@OptIn(ExperimentalComposeUiApi::class)
@Composable
fun AutofillLoginForm() {
    var email by remember { mutableStateOf("") }
    var password by remember { mutableStateOf("") }

    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        OutlinedTextField(
            value = email,
            onValueChange = { email = it },
            label = { Text("メール") },
            modifier = Modifier.fillMaxWidth().semantics {
                contentType = ContentType.EmailAddress
            },
            keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Email)
        )

        OutlinedTextField(
            value = password,
            onValueChange = { password = it },
            label = { Text("パスワード") },
            modifier = Modifier.fillMaxWidth().semantics {
                contentType = ContentType.Password
            },
            visualTransformation = PasswordVisualTransformation(),
            keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Password)
        )

        Button(onClick = {}, Modifier.fillMaxWidth()) { Text("ログイン") }
    }
}
```

---

## 新規登録フォーム

```kotlin
@OptIn(ExperimentalComposeUiApi::class)
@Composable
fun SignUpForm() {
    var name by remember { mutableStateOf("") }
    var email by remember { mutableStateOf("") }
    var password by remember { mutableStateOf("") }
    var phone by remember { mutableStateOf("") }

    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        OutlinedTextField(value = name, onValueChange = { name = it }, label = { Text("名前") },
            modifier = Modifier.fillMaxWidth().semantics { contentType = ContentType.PersonFullName })

        OutlinedTextField(value = email, onValueChange = { email = it }, label = { Text("メール") },
            modifier = Modifier.fillMaxWidth().semantics { contentType = ContentType.EmailAddress },
            keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Email))

        OutlinedTextField(value = phone, onValueChange = { phone = it }, label = { Text("電話番号") },
            modifier = Modifier.fillMaxWidth().semantics { contentType = ContentType.PhoneNumber },
            keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Phone))

        OutlinedTextField(value = password, onValueChange = { password = it }, label = { Text("パスワード") },
            modifier = Modifier.fillMaxWidth().semantics { contentType = ContentType.NewPassword },
            visualTransformation = PasswordVisualTransformation())

        Button(onClick = {}, Modifier.fillMaxWidth()) { Text("登録") }
    }
}
```

---

## 住所フォーム

```kotlin
@OptIn(ExperimentalComposeUiApi::class)
@Composable
fun AddressForm() {
    var postalCode by remember { mutableStateOf("") }
    var address by remember { mutableStateOf("") }

    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        OutlinedTextField(value = postalCode, onValueChange = { postalCode = it },
            label = { Text("郵便番号") },
            modifier = Modifier.fillMaxWidth().semantics { contentType = ContentType.PostalCode })

        OutlinedTextField(value = address, onValueChange = { address = it },
            label = { Text("住所") },
            modifier = Modifier.fillMaxWidth().semantics { contentType = ContentType.PostalAddress })
    }
}
```

---

## まとめ

| ContentType | 用途 |
|------------|------|
| `EmailAddress` | メールアドレス |
| `Password` | パスワード（ログイン） |
| `NewPassword` | 新規パスワード |
| `PersonFullName` | 氏名 |
| `PhoneNumber` | 電話番号 |

- `semantics { contentType = ... }`でオートフィルヒントを設定
- パスワードマネージャーが自動的にフォームを認識
- `Password`と`NewPassword`を使い分ける
- Googleパスワードマネージャー等と連携

---

8種類のAndroidアプリテンプレート（フォーム対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose SecureTextField](https://zenn.dev/myougatheaxo/articles/android-compose-compose-secure-text-field-2026)
- [Compose CredentialManager](https://zenn.dev/myougatheaxo/articles/android-compose-compose-credential-manager-2026)
- [Compose FocusRequester](https://zenn.dev/myougatheaxo/articles/android-compose-compose-focus-requester-2026)
