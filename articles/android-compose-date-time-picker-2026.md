---
title: "日付・時刻ピッカー実装ガイド — ComposeでDatePicker/TimePickerを使う"
emoji: "📅"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

Material3の**DatePicker**と**TimePicker**をComposeで実装する方法を解説します。

---

## DatePickerDialog

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun DatePickerExample() {
    var showDatePicker by remember { mutableStateOf(false) }
    var selectedDate by remember { mutableStateOf<Long?>(null) }

    val dateFormatter = remember { SimpleDateFormat("yyyy/MM/dd", Locale.JAPAN) }

    Column(Modifier.padding(16.dp)) {
        OutlinedTextField(
            value = selectedDate?.let { dateFormatter.format(Date(it)) } ?: "",
            onValueChange = {},
            readOnly = true,
            label = { Text("日付を選択") },
            trailingIcon = {
                IconButton(onClick = { showDatePicker = true }) {
                    Icon(Icons.Default.CalendarMonth, "日付選択")
                }
            },
            modifier = Modifier.fillMaxWidth()
        )
    }

    if (showDatePicker) {
        val datePickerState = rememberDatePickerState()

        DatePickerDialog(
            onDismissRequest = { showDatePicker = false },
            confirmButton = {
                TextButton(onClick = {
                    selectedDate = datePickerState.selectedDateMillis
                    showDatePicker = false
                }) {
                    Text("OK")
                }
            },
            dismissButton = {
                TextButton(onClick = { showDatePicker = false }) {
                    Text("キャンセル")
                }
            }
        ) {
            DatePicker(state = datePickerState)
        }
    }
}
```

---

## 日付範囲制限

```kotlin
val datePickerState = rememberDatePickerState(
    initialSelectedDateMillis = System.currentTimeMillis(),
    selectableDates = object : SelectableDates {
        override fun isSelectableDate(utcTimeMillis: Long): Boolean {
            // 今日以降のみ選択可能
            return utcTimeMillis >= System.currentTimeMillis() - 86400000
        }
        override fun isSelectableYear(year: Int): Boolean {
            return year >= 2024
        }
    }
)
```

---

## DateRangePicker（範囲選択）

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun DateRangePickerExample() {
    var showPicker by remember { mutableStateOf(false) }
    val state = rememberDateRangePickerState()

    if (showPicker) {
        DatePickerDialog(
            onDismissRequest = { showPicker = false },
            confirmButton = {
                TextButton(onClick = {
                    val start = state.selectedStartDateMillis
                    val end = state.selectedEndDateMillis
                    // 範囲を使って処理
                    showPicker = false
                }) {
                    Text("OK")
                }
            }
        ) {
            DateRangePicker(state = state)
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
    var showTimePicker by remember { mutableStateOf(false) }
    val timePickerState = rememberTimePickerState(
        initialHour = 9,
        initialMinute = 0,
        is24Hour = true
    )

    Column(Modifier.padding(16.dp)) {
        OutlinedTextField(
            value = "%02d:%02d".format(timePickerState.hour, timePickerState.minute),
            onValueChange = {},
            readOnly = true,
            label = { Text("時刻を選択") },
            trailingIcon = {
                IconButton(onClick = { showTimePicker = true }) {
                    Icon(Icons.Default.Schedule, "時刻選択")
                }
            }
        )
    }

    if (showTimePicker) {
        AlertDialog(
            onDismissRequest = { showTimePicker = false },
            confirmButton = {
                TextButton(onClick = { showTimePicker = false }) {
                    Text("OK")
                }
            },
            dismissButton = {
                TextButton(onClick = { showTimePicker = false }) {
                    Text("キャンセル")
                }
            },
            text = {
                TimePicker(state = timePickerState)
            }
        )
    }
}
```

---

## TimeInput（入力式）

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun TimeInputExample() {
    val state = rememberTimePickerState(is24Hour = true)

    TimeInput(state = state)

    // state.hour, state.minute で値を取得
}
```

`TimePicker`はダイヤル式、`TimeInput`はキーボード入力式。

---

## まとめ

- `DatePickerDialog` + `DatePicker`で日付選択
- `DateRangePicker`で期間選択
- `SelectableDates`で選択可能な日付を制限
- `TimePicker`（ダイヤル式）/ `TimeInput`（入力式）で時刻選択
- `rememberDatePickerState` / `rememberTimePickerState`で状態管理

---

8種類のAndroidアプリテンプレート（日時選択UI設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ダイアログ完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-dialog-guide-2026)
- [入力フォーム完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-forms-input-2026)
- [Material3コンポーネントカタログ](https://zenn.dev/myougatheaxo/articles/compose-material3-components-2026)
