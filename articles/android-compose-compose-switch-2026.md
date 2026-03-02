---
title: "Switch完全ガイド — Switch/Checkbox/RadioButton/ToggleButton"
emoji: "🔘"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**Switch系UI**（Switch、Checkbox、RadioButton、ToggleButton）を解説します。

---

## Switch

```kotlin
@Composable
fun SwitchExamples() {
    Column(Modifier.padding(16.dp)) {
        // 基本Switch
        var darkMode by remember { mutableStateOf(false) }
        ListItem(
            headlineContent = { Text("ダークモード") },
            trailingContent = {
                Switch(checked = darkMode, onCheckedChange = { darkMode = it })
            }
        )

        // アイコン付きSwitch
        var notifications by remember { mutableStateOf(true) }
        ListItem(
            headlineContent = { Text("通知") },
            trailingContent = {
                Switch(
                    checked = notifications,
                    onCheckedChange = { notifications = it },
                    thumbContent = {
                        Icon(
                            if (notifications) Icons.Default.Check else Icons.Default.Close,
                            null,
                            modifier = Modifier.size(SwitchDefaults.IconSize)
                        )
                    }
                )
            }
        )
    }
}
```

---

## Checkbox

```kotlin
@Composable
fun CheckboxGroup() {
    val options = listOf("Kotlin", "Java", "Dart", "Swift")
    var selected by remember { mutableStateOf(setOf<String>()) }

    Column(Modifier.padding(16.dp)) {
        Text("使用言語を選択", style = MaterialTheme.typography.titleMedium)
        options.forEach { option ->
            Row(
                Modifier
                    .fillMaxWidth()
                    .clickable {
                        selected = if (option in selected) selected - option else selected + option
                    }
                    .padding(vertical = 4.dp),
                verticalAlignment = Alignment.CenterVertically
            ) {
                Checkbox(
                    checked = option in selected,
                    onCheckedChange = {
                        selected = if (option in selected) selected - option else selected + option
                    }
                )
                Spacer(Modifier.width(8.dp))
                Text(option)
            }
        }
    }
}
```

---

## RadioButton

```kotlin
@Composable
fun RadioGroup() {
    val options = listOf("日本語", "English", "中文")
    var selected by remember { mutableStateOf(options[0]) }

    Column(Modifier.padding(16.dp)) {
        Text("言語設定", style = MaterialTheme.typography.titleMedium)
        options.forEach { option ->
            Row(
                Modifier
                    .fillMaxWidth()
                    .selectable(
                        selected = selected == option,
                        onClick = { selected = option },
                        role = Role.RadioButton
                    )
                    .padding(vertical = 4.dp),
                verticalAlignment = Alignment.CenterVertically
            ) {
                RadioButton(selected = selected == option, onClick = null)
                Spacer(Modifier.width(8.dp))
                Text(option)
            }
        }
    }
}
```

---

## まとめ

| コンポーネント | 用途 |
|-------------|------|
| `Switch` | ON/OFF切り替え |
| `Checkbox` | 複数選択 |
| `RadioButton` | 単一選択 |
| `TriStateCheckbox` | 全選択/部分選択 |

- `Switch`で設定のON/OFF
- `Checkbox`で複数項目の選択
- `RadioButton`で排他的な単一選択
- `thumbContent`でSwitchにアイコン表示

---

8種類のAndroidアプリテンプレート（設定画面対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Chip](https://zenn.dev/myougatheaxo/articles/android-compose-compose-chip-2026)
- [Slider](https://zenn.dev/myougatheaxo/articles/android-compose-compose-slider-2026)
- [DataStore](https://zenn.dev/myougatheaxo/articles/android-compose-datastore-2026)
