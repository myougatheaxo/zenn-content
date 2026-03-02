---
title: "DateTimePicker完全ガイド — DatePicker/TimePicker/DateRangePicker"
emoji: "📅"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**DateTimePicker**（DatePicker、TimePicker、DateRangePicker、ダイアログ統合）を解説します。

---

## DatePickerDialog

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun DatePickerExample() {
    var showDialog by remember { mutableStateOf(false) }
    var selectedDate by remember { mutableStateOf<Long?>(null) }
    val datePickerState = rememberDatePickerState()

    Column(Modifier.padding(16.dp)) {
        OutlinedButton(onClick = { showDialog = true }) {
            Text(selectedDate?.let {
                SimpleDateFormat("yyyy/MM/dd", Locale.JAPAN).format(Date(it))
            } ?: "日付を選択")
        }

        if (showDialog) {
            DatePickerDialog(
                onDismissRequest = { showDialog = false },
                confirmButton = {
                    TextButton(onClick = {
                        selectedDate = datePickerState.selectedDateMillis
                        showDialog = false
                    }) { Text("OK") }
                },
                dismissButton = {
                    TextButton(onClick = { showDialog = false }) { Text("キャンセル") }
                }
            ) {
                DatePicker(state = datePickerState)
            }
        }
    }
}
```

---

## TimePicker

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun TimePickerExample() {
    var showDialog by remember { mutableStateOf(false) }
    val timePickerState = rememberTimePickerState(
        initialHour = 9,
        initialMinute = 0,
        is24Hour = true
    )

    Column(Modifier.padding(16.dp)) {
        OutlinedButton(onClick = { showDialog = true }) {
            Text("${timePickerState.hour}:${"%02d".format(timePickerState.minute)}")
        }

        if (showDialog) {
            AlertDialog(
                onDismissRequest = { showDialog = false },
                title = { Text("時刻を選択") },
                text = { TimePicker(state = timePickerState) },
                confirmButton = {
                    TextButton(onClick = { showDialog = false }) { Text("OK") }
                },
                dismissButton = {
                    TextButton(onClick = { showDialog = false }) { Text("キャンセル") }
                }
            )
        }
    }
}
```

---

## DateRangePicker

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun DateRangeExample() {
    var showDialog by remember { mutableStateOf(false) }
    val rangeState = rememberDateRangePickerState()
    val fmt = SimpleDateFormat("MM/dd", Locale.JAPAN)

    Column(Modifier.padding(16.dp)) {
        OutlinedButton(onClick = { showDialog = true }) {
            val start = rangeState.selectedStartDateMillis?.let { fmt.format(Date(it)) } ?: "開始"
            val end = rangeState.selectedEndDateMillis?.let { fmt.format(Date(it)) } ?: "終了"
            Text("$start ~ $end")
        }

        if (showDialog) {
            DatePickerDialog(
                onDismissRequest = { showDialog = false },
                confirmButton = {
                    TextButton(onClick = { showDialog = false }) { Text("OK") }
                }
            ) {
                DateRangePicker(
                    state = rangeState,
                    modifier = Modifier.height(500.dp)
                )
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `DatePicker` | 日付選択 |
| `TimePicker` | 時刻選択 |
| `DateRangePicker` | 期間選択 |
| `DatePickerDialog` | ダイアログ表示 |

- `DatePicker`でMaterial3準拠の日付選択UI
- `TimePicker`で24時間/12時間形式対応
- `DateRangePicker`で開始〜終了日の範囲選択
- `rememberDatePickerState`で選択状態を保持

---

8種類のAndroidアプリテンプレート（日時選択対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Dialog](https://zenn.dev/myougatheaxo/articles/android-compose-dialog-2026)
- [TextField/バリデーション](https://zenn.dev/myougatheaxo/articles/android-compose-textfield-validation-2026)
- [フォーカス管理](https://zenn.dev/myougatheaxo/articles/android-compose-compose-focus-2026)
