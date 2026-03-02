---
title: "Compose CheckboxGroup完全ガイド — Checkbox/TriStateCheckbox/グループ選択"
emoji: "☑️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**Compose CheckboxGroup**（Checkbox、TriStateCheckbox、グループ選択、親子チェック連動）を解説します。

---

## 基本Checkbox

```kotlin
@Composable
fun CheckboxDemo() {
    var agreed by remember { mutableStateOf(false) }

    Row(Modifier.fillMaxWidth().clickable { agreed = !agreed }.padding(16.dp),
        verticalAlignment = Alignment.CenterVertically) {
        Checkbox(checked = agreed, onCheckedChange = { agreed = it })
        Spacer(Modifier.width(8.dp))
        Text("利用規約に同意する")
    }
}
```

---

## チェックボックスグループ

```kotlin
@Composable
fun CheckboxGroupDemo() {
    val options = listOf("Android", "iOS", "Web", "Desktop")
    val selectedOptions = remember { mutableStateListOf<String>() }

    Column(Modifier.padding(16.dp)) {
        Text("対応プラットフォーム", style = MaterialTheme.typography.titleMedium)
        Spacer(Modifier.height(8.dp))

        options.forEach { option ->
            Row(Modifier.fillMaxWidth().clickable {
                if (option in selectedOptions) selectedOptions.remove(option)
                else selectedOptions.add(option)
            }.padding(vertical = 4.dp), verticalAlignment = Alignment.CenterVertically) {
                Checkbox(
                    checked = option in selectedOptions,
                    onCheckedChange = {
                        if (it) selectedOptions.add(option) else selectedOptions.remove(option)
                    }
                )
                Spacer(Modifier.width(8.dp))
                Text(option)
            }
        }
        Spacer(Modifier.height(8.dp))
        Text("選択: ${selectedOptions.joinToString(", ")}",
            style = MaterialTheme.typography.bodySmall)
    }
}
```

---

## 親子TriStateCheckbox

```kotlin
@Composable
fun TriStateCheckboxDemo() {
    val childStates = remember { mutableStateListOf(false, false, false) }
    val labels = listOf("読み取り", "書き込み", "実行")

    val parentState = when {
        childStates.all { it } -> ToggleableState.On
        childStates.none { it } -> ToggleableState.Off
        else -> ToggleableState.Indeterminate
    }

    Column(Modifier.padding(16.dp)) {
        // 親
        Row(Modifier.clickable {
            val newState = parentState != ToggleableState.On
            childStates.indices.forEach { childStates[it] = newState }
        }.padding(vertical = 4.dp), verticalAlignment = Alignment.CenterVertically) {
            TriStateCheckbox(state = parentState, onClick = {
                val newState = parentState != ToggleableState.On
                childStates.indices.forEach { childStates[it] = newState }
            })
            Spacer(Modifier.width(8.dp))
            Text("すべての権限", style = MaterialTheme.typography.titleMedium)
        }

        // 子
        labels.forEachIndexed { index, label ->
            Row(Modifier.padding(start = 32.dp).clickable { childStates[index] = !childStates[index] }
                .padding(vertical = 4.dp), verticalAlignment = Alignment.CenterVertically) {
                Checkbox(checked = childStates[index],
                    onCheckedChange = { childStates[index] = it })
                Spacer(Modifier.width(8.dp))
                Text(label)
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `Checkbox` | 単一チェックボックス |
| `TriStateCheckbox` | 3状態（On/Off/中間） |
| `ToggleableState` | 3状態の値 |
| `mutableStateListOf` | 複数選択状態管理 |

- `Checkbox`で単一項目のON/OFF
- `mutableStateListOf`で複数選択を管理
- `TriStateCheckbox`で親子連動（全選択/一部選択/未選択）
- Row全体に`clickable`でタップ領域を広げる

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose RadioGroup](https://zenn.dev/myougatheaxo/articles/android-compose-compose-radio-group-2026)
- [Compose Switch](https://zenn.dev/myougatheaxo/articles/android-compose-compose-material-switch-2026)
- [Compose SettingsScreen](https://zenn.dev/myougatheaxo/articles/android-compose-compose-settings-screen-2026)
