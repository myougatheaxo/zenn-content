---
title: "オートフィル完全ガイド — Autofill Framework/セマンティクス/パスワード管理"
emoji: "📋"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "auth"]
published: true
---

## この記事で学べること

**オートフィル**（Autofill Framework、Composeセマンティクス、パスワードマネージャー連携、カスタムAutofillタイプ）を解説します。

---

## Composeオートフィル対応

```kotlin
@Composable
fun AutofillLoginForm() {
    var email by remember { mutableStateOf("") }
    var password by remember { mutableStateOf("") }

    Column(Modifier.padding(16.dp)) {
        OutlinedTextField(
            value = email,
            onValueChange = { email = it },
            label = { Text("メールアドレス") },
            keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Email),
            modifier = Modifier
                .fillMaxWidth()
                .semantics {
                    contentType = ContentType.EmailAddress
                }
        )

        Spacer(Modifier.height(16.dp))

        OutlinedTextField(
            value = password,
            onValueChange = { password = it },
            label = { Text("パスワード") },
            visualTransformation = PasswordVisualTransformation(),
            keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Password),
            modifier = Modifier
                .fillMaxWidth()
                .semantics {
                    contentType = ContentType.Password
                }
        )
    }
}
```

---

## カスタムAutofillタイプ

```kotlin
@Composable
fun AutofillAddressForm() {
    var name by remember { mutableStateOf("") }
    var postalCode by remember { mutableStateOf("") }
    var address by remember { mutableStateOf("") }
    var phone by remember { mutableStateOf("") }

    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        OutlinedTextField(
            value = name,
            onValueChange = { name = it },
            label = { Text("氏名") },
            modifier = Modifier.fillMaxWidth().semantics {
                contentType = ContentType.PersonFullName
            }
        )

        OutlinedTextField(
            value = postalCode,
            onValueChange = { postalCode = it },
            label = { Text("郵便番号") },
            keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Number),
            modifier = Modifier.fillMaxWidth().semantics {
                contentType = ContentType.PostalCode
            }
        )

        OutlinedTextField(
            value = address,
            onValueChange = { address = it },
            label = { Text("住所") },
            modifier = Modifier.fillMaxWidth().semantics {
                contentType = ContentType.PostalAddress
            }
        )

        OutlinedTextField(
            value = phone,
            onValueChange = { phone = it },
            label = { Text("電話番号") },
            keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Phone),
            modifier = Modifier.fillMaxWidth().semantics {
                contentType = ContentType.PhoneNumber
            }
        )
    }
}
```

---

## Credential Manager連携

```kotlin
@Composable
fun SmartLoginButton(viewModel: AuthViewModel = hiltViewModel()) {
    val context = LocalContext.current

    Button(onClick = {
        viewModel.signInWithSavedCredentials(context as Activity)
    }) {
        Icon(Icons.Default.Key, null)
        Spacer(Modifier.width(8.dp))
        Text("保存されたパスワードでログイン")
    }
}
```

---

## まとめ

| タイプ | ContentType |
|--------|-------------|
| メール | `EmailAddress` |
| パスワード | `Password` |
| 氏名 | `PersonFullName` |
| 電話 | `PhoneNumber` |

- `semantics { contentType }`でAutofillタイプを指定
- パスワードマネージャーと自動連携
- 住所フォームは`PostalCode`/`PostalAddress`で補完
- Credential Managerで保存済み認証情報を活用

---

8種類のAndroidアプリテンプレート（認証フォーム対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Google Sign-In](https://zenn.dev/myougatheaxo/articles/android-compose-google-signin-2026)
- [Credential Manager](https://zenn.dev/myougatheaxo/articles/android-compose-credential-manager-2026)
- [TextField/バリデーション](https://zenn.dev/myougatheaxo/articles/android-compose-textfield-validation-2026)
