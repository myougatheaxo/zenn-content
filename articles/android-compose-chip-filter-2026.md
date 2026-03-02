---
title: "FilterChip・AssistChip実装ガイド — ComposeでフィルタリングUI"
emoji: "🏷️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

Material3の**FilterChip**、**AssistChip**、**InputChip**を使ったフィルタリングUI・タグUIの実装方法を解説します。

---

## FilterChip（単一選択）

```kotlin
@Composable
fun FilterChipExample() {
    val categories = listOf("すべて", "仕事", "買い物", "勉強", "運動")
    var selected by remember { mutableStateOf("すべて") }

    LazyRow(
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        contentPadding = PaddingValues(horizontal = 16.dp)
    ) {
        items(categories) { category ->
            FilterChip(
                selected = selected == category,
                onClick = { selected = category },
                label = { Text(category) },
                leadingIcon = if (selected == category) {
                    { Icon(Icons.Default.Check, null, Modifier.size(18.dp)) }
                } else null
            )
        }
    }
}
```

---

## 複数選択FilterChip

```kotlin
@Composable
fun MultiSelectChips() {
    val tags = listOf("Kotlin", "Compose", "Android", "Firebase", "Room")
    var selectedTags by remember { mutableStateOf(setOf<String>()) }

    FlowRow(
        modifier = Modifier.padding(16.dp),
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        verticalArrangement = Arrangement.spacedBy(4.dp)
    ) {
        tags.forEach { tag ->
            FilterChip(
                selected = tag in selectedTags,
                onClick = {
                    selectedTags = if (tag in selectedTags)
                        selectedTags - tag
                    else
                        selectedTags + tag
                },
                label = { Text(tag) },
                leadingIcon = if (tag in selectedTags) {
                    { Icon(Icons.Default.Check, null, Modifier.size(18.dp)) }
                } else null
            )
        }
    }
}
```

---

## InputChip（削除可能タグ）

```kotlin
@Composable
fun InputChipExample() {
    var tags by remember { mutableStateOf(listOf("Android", "Kotlin", "Compose")) }
    var newTag by remember { mutableStateOf("") }

    Column(Modifier.padding(16.dp)) {
        FlowRow(
            horizontalArrangement = Arrangement.spacedBy(8.dp),
            verticalArrangement = Arrangement.spacedBy(4.dp)
        ) {
            tags.forEach { tag ->
                InputChip(
                    selected = false,
                    onClick = {},
                    label = { Text(tag) },
                    trailingIcon = {
                        IconButton(
                            onClick = { tags = tags - tag },
                            modifier = Modifier.size(18.dp)
                        ) {
                            Icon(Icons.Default.Close, "削除")
                        }
                    }
                )
            }
        }

        Spacer(Modifier.height(8.dp))

        OutlinedTextField(
            value = newTag,
            onValueChange = { newTag = it },
            label = { Text("タグを追加") },
            keyboardActions = KeyboardActions(
                onDone = {
                    if (newTag.isNotBlank()) {
                        tags = tags + newTag.trim()
                        newTag = ""
                    }
                }
            ),
            keyboardOptions = KeyboardOptions(imeAction = ImeAction.Done)
        )
    }
}
```

---

## AssistChip（アクション）

```kotlin
@Composable
fun AssistChipExample() {
    Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
        AssistChip(
            onClick = { /* カレンダーを開く */ },
            label = { Text("カレンダーに追加") },
            leadingIcon = { Icon(Icons.Default.CalendarMonth, null, Modifier.size(18.dp)) }
        )
        AssistChip(
            onClick = { /* 地図を開く */ },
            label = { Text("地図で見る") },
            leadingIcon = { Icon(Icons.Default.Place, null, Modifier.size(18.dp)) }
        )
    }
}
```

---

## リストフィルタリングとの連携

```kotlin
@Composable
fun FilteredTaskList(viewModel: TaskViewModel) {
    val tasks by viewModel.tasks.collectAsStateWithLifecycle()
    var selectedFilter by remember { mutableStateOf("すべて") }
    val filters = listOf("すべて", "未完了", "完了", "重要")

    val filtered = remember(tasks, selectedFilter) {
        when (selectedFilter) {
            "未完了" -> tasks.filter { !it.isCompleted }
            "完了" -> tasks.filter { it.isCompleted }
            "重要" -> tasks.filter { it.priority > 0 }
            else -> tasks
        }
    }

    Column {
        LazyRow(
            contentPadding = PaddingValues(horizontal = 16.dp),
            horizontalArrangement = Arrangement.spacedBy(8.dp)
        ) {
            items(filters) { filter ->
                FilterChip(
                    selected = selectedFilter == filter,
                    onClick = { selectedFilter = filter },
                    label = { Text(filter) }
                )
            }
        }

        LazyColumn {
            items(filtered, key = { it.id }) { task ->
                TaskItem(task)
            }
        }
    }
}
```

---

## まとめ

- `FilterChip` — 単一/複数選択フィルタ
- `InputChip` — 削除可能なタグ
- `AssistChip` — アクション付きチップ
- `FlowRow`で自動改行レイアウト
- `LazyRow`で横スクロール配置
- `remember(items, filter)`でフィルタリング最適化

---

8種類のAndroidアプリテンプレート（フィルタUI設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Material3コンポーネントカタログ](https://zenn.dev/myougatheaxo/articles/compose-material3-components-2026)
- [SearchBar実装ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-search-bar-2026)
- [LazyColumn完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-lazycolumn-guide-2026)
