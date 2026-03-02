---
title: "TimePicker完全ガイド — TimePickerDialog/Input切替/24時間表示"
emoji: "🕐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**TimePicker**（TimePickerDialog、Input/Dial切替、24時間表示、状態管理）を解説します。

---

## TimePicker基本

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun TimePickerExample() {
    var showPicker by remember { mutableStateOf(false) }
    val timePickerState = rememberTimePickerState(
        initialHour = 9, initialMinute = 0, is24Hour = true
    )

    Column(Modifier.padding(16.dp)) {
        OutlinedButton(onClick = { showPicker = true }) {
            Icon(Icons.Default.Schedule, null)
            Spacer(Modifier.width(8.dp))
            Text("%02d:%02d".format(timePickerState.hour, timePickerState.minute))
        }

        if (showPicker) {
            TimePickerDialog(
                onDismiss = { showPicker = false },
                onConfirm = { showPicker = false }
            ) {
                TimePicker(state = timePickerState)
            }
        }
    }
}

@Composable
fun TimePickerDialog(
    onDismiss: () -> Unit,
    onConfirm: () -> Unit,
    content: @Composable () -> Unit
) {
    AlertDialog(
        onDismissRequest = onDismiss,
        confirmButton = { TextButton(onClick = onConfirm) { Text("OK") } },
        dismissButton = { TextButton(onClick = onDismiss) { Text("キャンセル") } },
        title = { Text("時刻を選択") },
        text = { content() }
    )
}
```

---

## Input/Dial切替

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun TimePickerWithToggle() {
    var showPicker by remember { mutableStateOf(false) }
    var useInput by remember { mutableStateOf(false) }
    val state = rememberTimePickerState(initialHour = 14, initialMinute = 30, is24Hour = true)

    if (showPicker) {
        AlertDialog(
            onDismissRequest = { showPicker = false },
            confirmButton = { TextButton(onClick = { showPicker = false }) { Text("OK") } },
            dismissButton = { TextButton(onClick = { showPicker = false }) { Text("キャンセル") } },
            title = {
                Row(Modifier.fillMaxWidth(), horizontalArrangement = Arrangement.SpaceBetween) {
                    Text("時刻を選択")
                    IconButton(onClick = { useInput = !useInput }) {
                        Icon(
                            if (useInput) Icons.Default.Schedule else Icons.Default.Keyboard,
                            contentDescription = "入力切替"
                        )
                    }
                }
            },
            text = {
                if (useInput) TimeInput(state = state) else TimePicker(state = state)
            }
        )
    }

    OutlinedButton(onClick = { showPicker = true }) {
        Text("%02d:%02d".format(state.hour, state.minute))
    }
}
```

---

## アラーム設定UI

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun AlarmSettingScreen() {
    val alarmState = rememberTimePickerState(initialHour = 7, initialMinute = 0, is24Hour = false)
    var isEnabled by remember { mutableStateOf(true) }
    var showPicker by remember { mutableStateOf(false) }
    val selectedDays = remember { mutableStateListOf("月", "火", "水", "木", "金") }

    Card(Modifier.fillMaxWidth().padding(16.dp)) {
        Column(Modifier.padding(16.dp)) {
            Row(verticalAlignment = Alignment.CenterVertically) {
                Column(Modifier.weight(1f).clickable { showPicker = true }) {
                    Text(
                        "%02d:%02d".format(alarmState.hour, alarmState.minute),
                        style = MaterialTheme.typography.displayMedium
                    )
                    Text("タップして変更", style = MaterialTheme.typography.bodySmall, color = Color.Gray)
                }
                Switch(checked = isEnabled, onCheckedChange = { isEnabled = it })
            }
            Spacer(Modifier.height(12.dp))
            Row(horizontalArrangement = Arrangement.spacedBy(4.dp)) {
                listOf("月", "火", "水", "木", "金", "土", "日").forEach { day ->
                    FilterChip(
                        selected = day in selectedDays,
                        onClick = { if (day in selectedDays) selectedDays.remove(day) else selectedDays.add(day) },
                        label = { Text(day) }
                    )
                }
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `TimePicker` | ダイアル式時刻選択 |
| `TimeInput` | テキスト入力式 |
| `rememberTimePickerState` | 状態管理 |
| `is24Hour` | 24時間表示切替 |

- `TimePicker`でMaterial3準拠の時刻選択
- `TimeInput`でテキスト入力切替可能
- `is24Hour`で12/24時間表示切替
- AlertDialogでダイアログ表示

---

8種類のAndroidアプリテンプレート（フォーム対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [DateTimePicker](https://zenn.dev/myougatheaxo/articles/android-compose-compose-datetime-picker-2026)
- [AlertDialog](https://zenn.dev/myougatheaxo/articles/android-compose-compose-alert-dialog-2026)
- [Switch](https://zenn.dev/myougatheaxo/articles/android-compose-compose-switch-2026)
