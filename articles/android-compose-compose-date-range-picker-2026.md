---
title: "DateRangePicker完全ガイド — 範囲選択/期間フィルタ/カレンダーUI"
emoji: "📅"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**DateRangePicker**（日付範囲選択、期間フィルタ、フォーマット表示）を解説します。

---

## DateRangePicker基本

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun DateRangePickerExample() {
    var showPicker by remember { mutableStateOf(false) }
    val dateRangeState = rememberDateRangePickerState()
    val formatter = remember { SimpleDateFormat("yyyy/MM/dd", Locale.JAPAN) }

    Column(Modifier.padding(16.dp)) {
        val startText = dateRangeState.selectedStartDateMillis?.let { formatter.format(Date(it)) } ?: "開始日"
        val endText = dateRangeState.selectedEndDateMillis?.let { formatter.format(Date(it)) } ?: "終了日"

        OutlinedButton(onClick = { showPicker = true }) {
            Icon(Icons.Default.DateRange, null)
            Spacer(Modifier.width(8.dp))
            Text("$startText 〜 $endText")
        }

        if (showPicker) {
            DatePickerDialog(
                onDismissRequest = { showPicker = false },
                confirmButton = { TextButton(onClick = { showPicker = false }) { Text("OK") } },
                dismissButton = { TextButton(onClick = { showPicker = false }) { Text("キャンセル") } }
            ) {
                DateRangePicker(
                    state = dateRangeState,
                    modifier = Modifier.height(500.dp)
                )
            }
        }
    }
}
```

---

## 期間フィルタ

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun PeriodFilterScreen(onFilter: (Long?, Long?) -> Unit) {
    var showPicker by remember { mutableStateOf(false) }
    val state = rememberDateRangePickerState(
        selectableDates = object : SelectableDates {
            override fun isSelectableDate(utcTimeMillis: Long): Boolean {
                return utcTimeMillis <= System.currentTimeMillis()
            }
        }
    )
    val formatter = remember { SimpleDateFormat("M/d", Locale.JAPAN) }

    Column(Modifier.padding(16.dp)) {
        Text("期間で絞り込み", style = MaterialTheme.typography.titleMedium)
        Spacer(Modifier.height(8.dp))

        Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            listOf("今週" to 7, "今月" to 30, "3ヶ月" to 90).forEach { (label, days) ->
                FilterChip(
                    selected = false,
                    onClick = {
                        val end = System.currentTimeMillis()
                        val start = end - days.toLong() * 24 * 60 * 60 * 1000
                        onFilter(start, end)
                    },
                    label = { Text(label) }
                )
            }
            FilterChip(
                selected = false,
                onClick = { showPicker = true },
                label = { Text("カスタム") },
                leadingIcon = { Icon(Icons.Default.DateRange, null, Modifier.size(18.dp)) }
            )
        }

        if (showPicker) {
            DatePickerDialog(
                onDismissRequest = { showPicker = false },
                confirmButton = {
                    TextButton(onClick = {
                        onFilter(state.selectedStartDateMillis, state.selectedEndDateMillis)
                        showPicker = false
                    }) { Text("適用") }
                },
                dismissButton = { TextButton(onClick = { showPicker = false }) { Text("キャンセル") } }
            ) {
                DateRangePicker(state = state, modifier = Modifier.height(500.dp))
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `DateRangePicker` | 範囲選択カレンダー |
| `rememberDateRangePickerState` | 状態管理 |
| `selectedStartDateMillis` | 開始日（UTC ms） |
| `SelectableDates` | 選択可能日制限 |

- `DateRangePicker`で開始日〜終了日を選択
- `SelectableDates`で未来日の制限等が可能
- `DatePickerDialog`でダイアログ表示
- 期間フィルタ機能に最適

---

8種類のAndroidアプリテンプレート（フォーム対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [DateTimePicker](https://zenn.dev/myougatheaxo/articles/android-compose-compose-datetime-picker-2026)
- [TimePicker](https://zenn.dev/myougatheaxo/articles/android-compose-compose-time-picker-2026)
- [Chip](https://zenn.dev/myougatheaxo/articles/android-compose-compose-chip-2026)
