---
title: "Compose TextFieldState完全ガイド — BasicTextField2/状態管理/入力フィルタ"
emoji: "📝"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**Compose TextFieldState**（BasicTextField2、rememberTextFieldState、InputTransformation、OutputTransformation）を解説します。

---

## 基本TextFieldState

```kotlin
@Composable
fun TextFieldStateDemo() {
    val state = rememberTextFieldState()

    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(8.dp)) {
        BasicTextField(
            state = state,
            modifier = Modifier
                .fillMaxWidth()
                .border(1.dp, Color.Gray, RoundedCornerShape(8.dp))
                .padding(12.dp),
            textStyle = MaterialTheme.typography.bodyLarge
        )

        Text("入力文字数: ${state.text.length}")
    }
}
```

---

## InputTransformation

```kotlin
// 最大文字数制限
val maxLengthTransformation = InputTransformation {
    if (length > 100) {
        revertAllChanges()
    }
}

// 数字のみ
val digitsOnlyTransformation = InputTransformation {
    if (!asCharSequence().all { it.isDigit() }) {
        revertAllChanges()
    }
}

@Composable
fun FilteredTextField() {
    val state = rememberTextFieldState()

    BasicTextField(
        state = state,
        inputTransformation = InputTransformation.maxLength(50)
            .then(InputTransformation {
                // カスタムフィルタチェーン
            }),
        modifier = Modifier
            .fillMaxWidth()
            .background(Color(0xFFF5F5F5), RoundedCornerShape(8.dp))
            .padding(12.dp)
    )
}
```

---

## OutputTransformation（表示変換）

```kotlin
// 電話番号フォーマット表示
val phoneOutputTransformation = OutputTransformation {
    if (length >= 7) {
        insert(3, "-")
        insert(8, "-")
    } else if (length >= 3) {
        insert(3, "-")
    }
}

@Composable
fun PhoneField() {
    val state = rememberTextFieldState()

    BasicTextField(
        state = state,
        inputTransformation = InputTransformation.maxLength(11).then(
            InputTransformation {
                if (!asCharSequence().all { it.isDigit() }) revertAllChanges()
            }
        ),
        outputTransformation = phoneOutputTransformation,
        modifier = Modifier
            .fillMaxWidth()
            .border(1.dp, Color.Gray, RoundedCornerShape(8.dp))
            .padding(12.dp),
        textStyle = MaterialTheme.typography.bodyLarge
    )
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `rememberTextFieldState` | 状態保持 |
| `InputTransformation` | 入力制限 |
| `OutputTransformation` | 表示変換 |
| `BasicTextField(state=)` | 新API使用 |

- `TextFieldState`はCompose新世代のテキスト入力API
- `InputTransformation`で入力時にフィルタリング
- `OutputTransformation`で表示時にフォーマット変換
- `then()`でフィルタチェーンを構築

---

8種類のAndroidアプリテンプレート（フォーム対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose FocusRequester](https://zenn.dev/myougatheaxo/articles/android-compose-compose-focus-requester-2026)
- [Compose InputFilter](https://zenn.dev/myougatheaxo/articles/android-compose-compose-input-filter-2026)
- [Compose SecureTextField](https://zenn.dev/myougatheaxo/articles/android-compose-compose-secure-text-field-2026)
