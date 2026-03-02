---
title: "パスワード強度チェッカー実装 — Composeでバリデーション付き入力UI"
emoji: "🔑"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "security"]
published: true
---

## この記事で学べること

Composeで**パスワード強度インジケーター**付きの入力フォームを実装する方法を解説します。

---

## パスワード強度の判定

```kotlin
enum class PasswordStrength(val label: String, val color: Color) {
    WEAK("弱い", Color(0xFFE53935)),
    FAIR("まあまあ", Color(0xFFFF9800)),
    GOOD("良い", Color(0xFF4CAF50)),
    STRONG("強い", Color(0xFF1B5E20))
}

fun checkPasswordStrength(password: String): PasswordStrength {
    var score = 0
    if (password.length >= 8) score++
    if (password.length >= 12) score++
    if (password.any { it.isUpperCase() }) score++
    if (password.any { it.isDigit() }) score++
    if (password.any { !it.isLetterOrDigit() }) score++

    return when {
        score <= 1 -> PasswordStrength.WEAK
        score == 2 -> PasswordStrength.FAIR
        score == 3 -> PasswordStrength.GOOD
        else -> PasswordStrength.STRONG
    }
}
```

---

## パスワード入力フィールド

```kotlin
@Composable
fun PasswordField() {
    var password by rememberSaveable { mutableStateOf("") }
    var isVisible by remember { mutableStateOf(false) }
    val strength = remember(password) { checkPasswordStrength(password) }

    Column(Modifier.padding(16.dp)) {
        OutlinedTextField(
            value = password,
            onValueChange = { password = it },
            label = { Text("パスワード") },
            visualTransformation = if (isVisible) VisualTransformation.None
                                   else PasswordVisualTransformation(),
            trailingIcon = {
                IconButton(onClick = { isVisible = !isVisible }) {
                    Icon(
                        if (isVisible) Icons.Default.VisibilityOff
                        else Icons.Default.Visibility,
                        if (isVisible) "隠す" else "表示"
                    )
                }
            },
            modifier = Modifier.fillMaxWidth()
        )

        if (password.isNotEmpty()) {
            Spacer(Modifier.height(8.dp))

            // 強度バー
            LinearProgressIndicator(
                progress = {
                    when (strength) {
                        PasswordStrength.WEAK -> 0.25f
                        PasswordStrength.FAIR -> 0.5f
                        PasswordStrength.GOOD -> 0.75f
                        PasswordStrength.STRONG -> 1f
                    }
                },
                modifier = Modifier.fillMaxWidth().height(4.dp).clip(RoundedCornerShape(2.dp)),
                color = strength.color,
                trackColor = Color.LightGray
            )

            Spacer(Modifier.height(4.dp))
            Text(
                strength.label,
                color = strength.color,
                style = MaterialTheme.typography.bodySmall
            )

            // 要件チェックリスト
            Spacer(Modifier.height(8.dp))
            PasswordRequirement("8文字以上", password.length >= 8)
            PasswordRequirement("大文字を含む", password.any { it.isUpperCase() })
            PasswordRequirement("数字を含む", password.any { it.isDigit() })
            PasswordRequirement("記号を含む", password.any { !it.isLetterOrDigit() })
        }
    }
}

@Composable
fun PasswordRequirement(text: String, isMet: Boolean) {
    Row(verticalAlignment = Alignment.CenterVertically) {
        Icon(
            if (isMet) Icons.Default.Check else Icons.Default.Close,
            null,
            tint = if (isMet) Color(0xFF4CAF50) else Color.Gray,
            modifier = Modifier.size(16.dp)
        )
        Spacer(Modifier.width(4.dp))
        Text(
            text,
            style = MaterialTheme.typography.bodySmall,
            color = if (isMet) MaterialTheme.colorScheme.onSurface else Color.Gray
        )
    }
}
```

---

## まとめ

- スコアベースの強度判定（長さ・大文字・数字・記号）
- `LinearProgressIndicator`で視覚的な強度表示
- 目アイコンでパスワードの表示/非表示切り替え
- チェックリストで要件の充足状況を表示
- `rememberSaveable`で画面回転対応

---

8種類のAndroidアプリテンプレート（セキュリティ設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [入力フォーム完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-forms-input-2026)
- [Firebase Authentication実装](https://zenn.dev/myougatheaxo/articles/android-firebase-auth-2026)
- [セキュリティチェックリスト](https://zenn.dev/myougatheaxo/articles/android-security-checklist-2026)
