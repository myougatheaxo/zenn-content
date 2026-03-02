---
title: "Compose DatePicker完全ガイド — M3 DatePicker/DateRangePicker/カスタム制約"
emoji: "📅"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**Compose DatePicker**（M3 DatePicker、DatePickerDialog、DateRangePicker、日付制約）を解説します。

---

## 基本DatePicker

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun DatePickerDemo() {
    var showDialog by remember { mutableStateOf(false) }
    val datePickerState = rememberDatePickerState()

    Column(Modifier.padding(16.dp)) {
        val selectedDate = datePickerState.selectedDateMillis?.let {
            SimpleDateFormat("yyyy/MM/dd", Locale.getDefault()).format(Date(it))
        } ?: "未選択"

        OutlinedButton(onClick = { showDialog = true }) {
            Icon(Icons.Default.CalendarToday, null)
            Spacer(Modifier.width(8.dp))
            Text("日付: $selectedDate")
        }

        if (showDialog) {
            DatePickerDialog(
                onDismissRequest = { showDialog = false },
                confirmButton = {
                    TextButton(onClick = { showDialog = false }) { Text("OK") }
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

## DateRangePicker

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun DateRangePickerDemo() {
    var showDialog by remember { mutableStateOf(false) }
    val dateRangePickerState = rememberDateRangePickerState()

    Column(Modifier.padding(16.dp)) {
        val format = SimpleDateFormat("MM/dd", Locale.getDefault())
        val start = dateRangePickerState.selectedStartDateMillis?.let { format.format(Date(it)) }
        val end = dateRangePickerState.selectedEndDateMillis?.let { format.format(Date(it)) }
        val range = if (start != null && end != null) "$start - $end" else "期間を選択"

        OutlinedButton(onClick = { showDialog = true }) { Text(range) }

        if (showDialog) {
            DatePickerDialog(
                onDismissRequest = { showDialog = false },
                confirmButton = {
                    TextButton(onClick = { showDialog = false }) { Text("OK") }
                }
            ) {
                DateRangePicker(state = dateRangePickerState, Modifier.height(500.dp))
            }
        }
    }
}
```

---

## 日付制約

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun ConstrainedDatePicker() {
    val today = System.currentTimeMillis()
    val oneYearLater = today + 365L * 24 * 60 * 60 * 1000

    val datePickerState = rememberDatePickerState(
        initialSelectedDateMillis = today,
        selectableDates = object : SelectableDates {
            override fun isSelectableDate(utcTimeMillis: Long): Boolean {
                return utcTimeMillis in today..oneYearLater
            }
            override fun isSelectableYear(year: Int): Boolean {
                val currentYear = Calendar.getInstance().get(Calendar.YEAR)
                return year in currentYear..currentYear + 1
            }
        }
    )

    DatePicker(state = datePickerState)
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `DatePicker` | 単一日付選択 |
| `DateRangePicker` | 範囲日付選択 |
| `DatePickerDialog` | ダイアログ表示 |
| `SelectableDates` | 日付制約 |

- `rememberDatePickerState()`で状態管理
- `DatePickerDialog`でモーダル表示
- `SelectableDates`で選択可能日を制限
- `selectedDateMillis`はUTCミリ秒（変換注意）

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose TimePicker](https://zenn.dev/myougatheaxo/articles/android-compose-compose-time-picker-2026)
- [Compose DateRangePicker](https://zenn.dev/myougatheaxo/articles/android-compose-compose-date-range-picker-2026)
- [Compose Dialog](https://zenn.dev/myougatheaxo/articles/android-compose-compose-dialog-2026)
