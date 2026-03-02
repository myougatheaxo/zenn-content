---
title: "VisualTransformation/InputFilter完全ガイド — 入力マスク/フォーマット/フィルタリング"
emoji: "🔤"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "textfield"]
published: true
---

## この記事で学べること

**VisualTransformation/入力フィルタ**（電話番号マスク、クレジットカードフォーマット、入力制限）を解説します。

---

## VisualTransformation

```kotlin
// 電話番号フォーマット: 090-1234-5678
class PhoneVisualTransformation : VisualTransformation {
    override fun filter(text: AnnotatedString): TransformedText {
        val trimmed = text.text.take(11)
        val out = buildString {
            trimmed.forEachIndexed { i, c ->
                append(c)
                if (i == 2 || i == 6) append('-')
            }
        }
        return TransformedText(
            AnnotatedString(out),
            object : OffsetMapping {
                override fun originalToTransformed(offset: Int): Int {
                    if (offset <= 2) return offset
                    if (offset <= 6) return offset + 1
                    return offset + 2
                }
                override fun transformedToOriginal(offset: Int): Int {
                    if (offset <= 3) return offset
                    if (offset <= 8) return offset - 1
                    return offset - 2
                }
            }
        )
    }
}

@Composable
fun PhoneInput() {
    var phone by remember { mutableStateOf("") }
    OutlinedTextField(
        value = phone,
        onValueChange = { phone = it.filter { c -> c.isDigit() }.take(11) },
        label = { Text("電話番号") },
        visualTransformation = PhoneVisualTransformation(),
        keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Number),
        modifier = Modifier.fillMaxWidth()
    )
}
```

---

## クレジットカード

```kotlin
class CreditCardTransformation : VisualTransformation {
    override fun filter(text: AnnotatedString): TransformedText {
        val trimmed = text.text.take(16)
        val out = buildString {
            trimmed.forEachIndexed { i, c ->
                if (i > 0 && i % 4 == 0) append(' ')
                append(c)
            }
        }
        return TransformedText(
            AnnotatedString(out),
            object : OffsetMapping {
                override fun originalToTransformed(offset: Int) = offset + offset / 4
                override fun transformedToOriginal(offset: Int) = offset - offset / 5
            }
        )
    }
}

@Composable
fun CreditCardInput() {
    var card by remember { mutableStateOf("") }
    OutlinedTextField(
        value = card,
        onValueChange = { card = it.filter { c -> c.isDigit() }.take(16) },
        label = { Text("カード番号") },
        visualTransformation = CreditCardTransformation(),
        keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Number),
        modifier = Modifier.fillMaxWidth()
    )
}
```

---

## 入力フィルタリング

```kotlin
@Composable
fun FilteredInputExamples() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(8.dp)) {
        // 数字のみ
        var number by remember { mutableStateOf("") }
        OutlinedTextField(
            value = number,
            onValueChange = { number = it.filter { c -> c.isDigit() } },
            label = { Text("数字のみ") },
            modifier = Modifier.fillMaxWidth()
        )

        // 英字のみ
        var alpha by remember { mutableStateOf("") }
        OutlinedTextField(
            value = alpha,
            onValueChange = { alpha = it.filter { c -> c.isLetter() } },
            label = { Text("英字のみ") },
            modifier = Modifier.fillMaxWidth()
        )

        // 最大文字数
        var limited by remember { mutableStateOf("") }
        OutlinedTextField(
            value = limited,
            onValueChange = { if (it.length <= 20) limited = it },
            label = { Text("最大20文字") },
            supportingText = { Text("${limited.length}/20") },
            modifier = Modifier.fillMaxWidth()
        )
    }
}
```

---

## まとめ

| 機能 | 用途 |
|------|------|
| `VisualTransformation` | 表示のみ変換 |
| `OffsetMapping` | カーソル位置対応 |
| `onValueChange`フィルタ | 入力値制限 |
| `PasswordVisualTransformation` | パスワード表示 |

- `VisualTransformation`で表示フォーマットを変更
- `OffsetMapping`でカーソル位置を正しくマッピング
- `onValueChange`で入力値をフィルタリング
- 電話番号/クレジットカード/郵便番号に活用

---

8種類のAndroidアプリテンプレート（フォーム対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [KeyboardOptions](https://zenn.dev/myougatheaxo/articles/android-compose-compose-keyboard-options-2026)
- [TextField/バリデーション](https://zenn.dev/myougatheaxo/articles/android-compose-textfield-validation-2026)
- [フォーカス管理](https://zenn.dev/myougatheaxo/articles/android-compose-compose-focus-2026)
