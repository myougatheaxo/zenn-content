---
title: "Compose入力フォーム完全ガイド — TextField・バリデーション・送信パターン"
emoji: "📝"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "forms"]
published: true
---

## この記事で学べること

アプリに入力フォームは不可欠です。Jetpack Composeでの**TextField、バリデーション、フォーム送信**のパターンを解説します。

---

## 基本のTextField

```kotlin
@Composable
fun BasicInput() {
    var text by remember { mutableStateOf("") }

    TextField(
        value = text,
        onValueChange = { text = it },
        label = { Text("名前") },
        placeholder = { Text("入力してください") }
    )
}
```

### OutlinedTextField（推奨）

```kotlin
OutlinedTextField(
    value = text,
    onValueChange = { text = it },
    label = { Text("メールアドレス") },
    leadingIcon = { Icon(Icons.Default.Email, contentDescription = null) },
    keyboardOptions = KeyboardOptions(
        keyboardType = KeyboardType.Email,
        imeAction = ImeAction.Next
    ),
    singleLine = true,
    modifier = Modifier.fillMaxWidth()
)
```

---

## バリデーション

```kotlin
@Composable
fun ValidatedInput() {
    var email by remember { mutableStateOf("") }
    var isError by remember { mutableStateOf(false) }

    OutlinedTextField(
        value = email,
        onValueChange = {
            email = it
            isError = it.isNotEmpty() && !it.contains("@")
        },
        label = { Text("メールアドレス") },
        isError = isError,
        supportingText = {
            if (isError) {
                Text("有効なメールアドレスを入力してください")
            }
        },
        modifier = Modifier.fillMaxWidth()
    )
}
```

### 複数フィールドのバリデーション

```kotlin
data class FormState(
    val name: String = "",
    val email: String = "",
    val amount: String = ""
) {
    val isValid: Boolean
        get() = name.isNotBlank()
            && email.contains("@")
            && amount.toIntOrNull() != null
            && (amount.toIntOrNull() ?: 0) > 0
}
```

---

## 数値入力

```kotlin
OutlinedTextField(
    value = amount,
    onValueChange = { newValue ->
        // 数字のみ許可
        if (newValue.all { it.isDigit() }) {
            amount = newValue
        }
    },
    label = { Text("金額") },
    prefix = { Text("¥") },
    keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Number),
    singleLine = true
)
```

---

## DropdownMenu

```kotlin
@Composable
fun CategoryPicker(
    categories: List<String>,
    selected: String,
    onSelect: (String) -> Unit
) {
    var expanded by remember { mutableStateOf(false) }

    ExposedDropdownMenuBox(
        expanded = expanded,
        onExpandedChange = { expanded = !expanded }
    ) {
        OutlinedTextField(
            value = selected,
            onValueChange = {},
            readOnly = true,
            trailingIcon = { ExposedDropdownMenuDefaults.TrailingIcon(expanded) },
            modifier = Modifier
                .menuAnchor()
                .fillMaxWidth()
        )
        ExposedDropdownMenu(
            expanded = expanded,
            onDismissRequest = { expanded = false }
        ) {
            categories.forEach { category ->
                DropdownMenuItem(
                    text = { Text(category) },
                    onClick = {
                        onSelect(category)
                        expanded = false
                    }
                )
            }
        }
    }
}
```

---

## DatePicker

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun DatePickerField(
    selectedDate: Long?,
    onDateSelected: (Long) -> Unit
) {
    var showPicker by remember { mutableStateOf(false) }
    val datePickerState = rememberDatePickerState(initialSelectedDateMillis = selectedDate)

    OutlinedTextField(
        value = selectedDate?.let {
            SimpleDateFormat("yyyy/MM/dd", Locale.getDefault()).format(Date(it))
        } ?: "",
        onValueChange = {},
        readOnly = true,
        label = { Text("日付") },
        trailingIcon = {
            IconButton(onClick = { showPicker = true }) {
                Icon(Icons.Default.DateRange, "日付選択")
            }
        },
        modifier = Modifier.fillMaxWidth()
    )

    if (showPicker) {
        DatePickerDialog(
            onDismissRequest = { showPicker = false },
            confirmButton = {
                TextButton(onClick = {
                    datePickerState.selectedDateMillis?.let { onDateSelected(it) }
                    showPicker = false
                }) { Text("OK") }
            }
        ) {
            DatePicker(state = datePickerState)
        }
    }
}
```

---

## フォーム送信パターン

```kotlin
@Composable
fun AddExpenseForm(viewModel: ExpenseViewModel) {
    var amount by remember { mutableStateOf("") }
    var category by remember { mutableStateOf("食費") }
    var note by remember { mutableStateOf("") }

    Column(Modifier.padding(16.dp)) {
        OutlinedTextField(
            value = amount,
            onValueChange = { if (it.all { c -> c.isDigit() }) amount = it },
            label = { Text("金額") },
            prefix = { Text("¥") },
            keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Number),
            modifier = Modifier.fillMaxWidth()
        )

        Spacer(Modifier.height(8.dp))

        // CategoryPicker(...)

        Spacer(Modifier.height(8.dp))

        OutlinedTextField(
            value = note,
            onValueChange = { note = it },
            label = { Text("メモ（任意）") },
            modifier = Modifier.fillMaxWidth()
        )

        Spacer(Modifier.height(16.dp))

        Button(
            onClick = {
                viewModel.addExpense(
                    amount = amount.toIntOrNull() ?: 0,
                    category = category,
                    note = note
                )
                amount = ""
                note = ""
            },
            enabled = amount.isNotBlank() && (amount.toIntOrNull() ?: 0) > 0,
            modifier = Modifier.fillMaxWidth()
        ) {
            Text("保存")
        }
    }
}
```

---

## まとめ

- `OutlinedTextField`が基本（Material3推奨）
- `isError` + `supportingText`でバリデーション表示
- `keyboardOptions`で入力タイプを制御
- `ExposedDropdownMenuBox`でドロップダウン
- `DatePickerDialog`で日付選択
- フォーム全体の`isValid`でボタンの`enabled`を制御

---

8種類のAndroidアプリテンプレート（入力フォーム実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Composeレイアウト入門](https://zenn.dev/myougatheaxo/articles/compose-layout-basics-2026)
- [ViewModel + StateFlow入門](https://zenn.dev/myougatheaxo/articles/android-viewmodel-stateflow-2026)
- [Material3テーマ完全ガイド](https://zenn.dev/myougatheaxo/articles/material3-theme-guide-2026)
