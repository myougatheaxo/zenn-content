---
title: "Chip完全ガイド — AssistChip/FilterChip/InputChip/SuggestionChip"
emoji: "🏷️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**Chip**（AssistChip、FilterChip、InputChip、SuggestionChip）を解説します。

---

## Chipの種類

```kotlin
@Composable
fun ChipShowcase() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        // AssistChip
        AssistChip(
            onClick = {},
            label = { Text("ルート検索") },
            leadingIcon = { Icon(Icons.Default.Directions, null, Modifier.size(18.dp)) }
        )

        // FilterChip
        var selected by remember { mutableStateOf(false) }
        FilterChip(
            selected = selected,
            onClick = { selected = !selected },
            label = { Text("Kotlin") },
            leadingIcon = if (selected) {{ Icon(Icons.Default.Check, null, Modifier.size(18.dp)) }} else null
        )

        // InputChip
        var showChip by remember { mutableStateOf(true) }
        if (showChip) {
            InputChip(
                selected = false,
                onClick = {},
                label = { Text("user@example.com") },
                trailingIcon = {
                    IconButton(onClick = { showChip = false }, Modifier.size(18.dp)) {
                        Icon(Icons.Default.Close, "削除")
                    }
                }
            )
        }

        // SuggestionChip
        SuggestionChip(
            onClick = {},
            label = { Text("おすすめ: Jetpack Compose") }
        )
    }
}
```

---

## フィルターChipグループ

```kotlin
@Composable
fun FilterChipGroup() {
    val categories = listOf("全て", "Kotlin", "Compose", "Room", "Hilt", "Navigation")
    var selectedCategories by remember { mutableStateOf(setOf("全て")) }

    FlowRow(
        Modifier.padding(16.dp),
        horizontalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        categories.forEach { category ->
            FilterChip(
                selected = category in selectedCategories,
                onClick = {
                    selectedCategories = if (category == "全て") {
                        setOf("全て")
                    } else {
                        val newSet = selectedCategories - "全て"
                        if (category in newSet) newSet - category else newSet + category
                    }.ifEmpty { setOf("全て") }
                },
                label = { Text(category) },
                leadingIcon = if (category in selectedCategories) {
                    { Icon(Icons.Default.Check, null, Modifier.size(18.dp)) }
                } else null
            )
        }
    }
}
```

---

## InputChip（タグ入力）

```kotlin
@Composable
fun TagInput() {
    var text by remember { mutableStateOf("") }
    var tags by remember { mutableStateOf(listOf<String>()) }

    Column(Modifier.padding(16.dp)) {
        FlowRow(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            tags.forEach { tag ->
                InputChip(
                    selected = false,
                    onClick = {},
                    label = { Text(tag) },
                    trailingIcon = {
                        IconButton(onClick = { tags = tags - tag }, Modifier.size(18.dp)) {
                            Icon(Icons.Default.Close, "削除")
                        }
                    }
                )
            }
        }
        Spacer(Modifier.height(8.dp))
        OutlinedTextField(
            value = text,
            onValueChange = { text = it },
            placeholder = { Text("タグを追加...") },
            keyboardOptions = KeyboardOptions(imeAction = ImeAction.Done),
            keyboardActions = KeyboardActions(onDone = {
                if (text.isNotBlank() && text !in tags) {
                    tags = tags + text.trim()
                    text = ""
                }
            }),
            modifier = Modifier.fillMaxWidth()
        )
    }
}
```

---

## まとめ

| Chip | 用途 |
|------|------|
| `AssistChip` | アクション提案 |
| `FilterChip` | フィルター選択 |
| `InputChip` | ユーザー入力表示 |
| `SuggestionChip` | サジェスト表示 |

- `FilterChip`で複数選択フィルターUI
- `InputChip`でタグ入力・削除
- `FlowRow`でChipを自動折り返し配置
- `leadingIcon`/`trailingIcon`でアイコン付きChip

---

8種類のAndroidアプリテンプレート（Material3 UI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [FlowRow](https://zenn.dev/myougatheaxo/articles/android-compose-compose-flow-row-2026)
- [Typography](https://zenn.dev/myougatheaxo/articles/android-compose-compose-typography-2026)
- [SearchBar](https://zenn.dev/myougatheaxo/articles/android-compose-compose-search-bar-2026)
