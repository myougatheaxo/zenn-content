---
title: "Compose SecureTextField完全ガイド — パスワード入力/表示切替/バリデーション"
emoji: "🔒"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "security"]
published: true
---

## この記事で学べること

**Compose SecureTextField**（パスワード入力、表示/非表示切替、強度バリデーション、セキュアな入力処理）を解説します。

---

## 基本パスワードフィールド

```kotlin
@Composable
fun PasswordField(
    password: String,
    onPasswordChange: (String) -> Unit,
    modifier: Modifier = Modifier
) {
    var passwordVisible by remember { mutableStateOf(false) }

    OutlinedTextField(
        value = password,
        onValueChange = onPasswordChange,
        label = { Text("パスワード") },
        modifier = modifier.fillMaxWidth(),
        singleLine = true,
        visualTransformation = if (passwordVisible)
            VisualTransformation.None
        else PasswordVisualTransformation(),
        keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Password),
        trailingIcon = {
            IconButton(onClick = { passwordVisible = !passwordVisible }) {
                Icon(
                    imageVector = if (passwordVisible) Icons.Default.VisibilityOff
                    else Icons.Default.Visibility,
                    contentDescription = if (passwordVisible) "非表示" else "表示"
                )
            }
        }
    )
}
```

---

## パスワード強度チェック

```kotlin
enum class PasswordStrength(val label: String, val color: Color) {
    WEAK("弱い", Color.Red),
    MEDIUM("普通", Color(0xFFFF9800)),
    STRONG("強い", Color(0xFF4CAF50))
}

fun checkPasswordStrength(password: String): PasswordStrength {
    val hasUpper = password.any { it.isUpperCase() }
    val hasLower = password.any { it.isLowerCase() }
    val hasDigit = password.any { it.isDigit() }
    val hasSpecial = password.any { !it.isLetterOrDigit() }
    val score = listOf(hasUpper, hasLower, hasDigit, hasSpecial, password.length >= 8).count { it }
    return when {
        score >= 4 -> PasswordStrength.STRONG
        score >= 2 -> PasswordStrength.MEDIUM
        else -> PasswordStrength.WEAK
    }
}

@Composable
fun PasswordWithStrength() {
    var password by remember { mutableStateOf("") }
    val strength = checkPasswordStrength(password)

    Column(verticalArrangement = Arrangement.spacedBy(8.dp)) {
        PasswordField(password = password, onPasswordChange = { password = it })

        if (password.isNotEmpty()) {
            LinearProgressIndicator(
                progress = { when (strength) { PasswordStrength.WEAK -> 0.33f; PasswordStrength.MEDIUM -> 0.66f; PasswordStrength.STRONG -> 1f } },
                modifier = Modifier.fillMaxWidth().height(4.dp),
                color = strength.color
            )
            Text(strength.label, color = strength.color, style = MaterialTheme.typography.bodySmall)
        }
    }
}
```

---

## パスワード確認付きフォーム

```kotlin
@Composable
fun PasswordConfirmForm(onSubmit: (String) -> Unit) {
    var password by remember { mutableStateOf("") }
    var confirmPassword by remember { mutableStateOf("") }
    val isMatch = password == confirmPassword && password.isNotEmpty()

    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        PasswordField(password = password, onPasswordChange = { password = it })

        OutlinedTextField(
            value = confirmPassword,
            onValueChange = { confirmPassword = it },
            label = { Text("パスワード確認") },
            modifier = Modifier.fillMaxWidth(),
            visualTransformation = PasswordVisualTransformation(),
            isError = confirmPassword.isNotEmpty() && !isMatch,
            supportingText = {
                if (confirmPassword.isNotEmpty() && !isMatch) Text("パスワードが一致しません")
            }
        )

        Button(onClick = { onSubmit(password) }, enabled = isMatch, modifier = Modifier.fillMaxWidth()) {
            Text("登録")
        }
    }
}
```

---

## まとめ

| 機能 | 実装 |
|------|------|
| パスワード非表示 | `PasswordVisualTransformation()` |
| 表示切替 | `trailingIcon` + `mutableStateOf` |
| 強度チェック | 正規表現/条件カウント |
| 確認フィールド | 一致バリデーション |

- `PasswordVisualTransformation`でマスク表示
- `KeyboardType.Password`で適切なキーボード表示
- パスワード強度インジケーターでUX向上
- 確認フィールドでミス入力防止

---

8種類のAndroidアプリテンプレート（認証画面対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Biometric](https://zenn.dev/myougatheaxo/articles/android-compose-compose-biometric-2026)
- [Compose FocusRequester](https://zenn.dev/myougatheaxo/articles/android-compose-compose-focus-requester-2026)
- [Compose TextFieldState](https://zenn.dev/myougatheaxo/articles/android-compose-compose-text-field-state-2026)
