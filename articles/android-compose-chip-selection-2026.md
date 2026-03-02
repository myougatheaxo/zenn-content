---
title: "Chip/FilterChipガイド — Composeでのタグ選択UI"
emoji: "🏷️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

Composeの**Chip系コンポーネント**（FilterChip、InputChip、SuggestionChip、FlowRow）を解説します。

---

## FilterChip

```kotlin
@Composable
fun FilterChipGroup() {
    val filters = listOf("Android", "iOS", "Web", "Backend")
    var selectedFilters by remember { mutableStateOf(setOf<String>()) }

    FlowRow(
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        modifier = Modifier.padding(16.dp)
    ) {
        filters.forEach { filter ->
            FilterChip(
                selected = filter in selectedFilters,
                onClick = {
                    selectedFilters = if (filter in selectedFilters) {
                        selectedFilters - filter
                    } else {
                        selectedFilters + filter
                    }
                },
                label = { Text(filter) },
                leadingIcon = if (filter in selectedFilters) {
                    { Icon(Icons.Default.Check, null, Modifier.size(18.dp)) }
                } else null
            )
        }
    }
}
```

---

## InputChip（削除可能）

```kotlin
@Composable
fun InputChipExample() {
    var tags by remember { mutableStateOf(listOf("Kotlin", "Compose", "Android")) }

    FlowRow(
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        modifier = Modifier.padding(16.dp)
    ) {
        tags.forEach { tag ->
            InputChip(
                selected = false,
                onClick = {},
                label = { Text(tag) },
                trailingIcon = {
                    Icon(
                        Icons.Default.Close,
                        contentDescription = "削除",
                        modifier = Modifier
                            .size(18.dp)
                            .clickable { tags = tags - tag }
                    )
                }
            )
        }
    }
}
```

---

## SuggestionChip

```kotlin
@Composable
fun SuggestionChipExample(onSuggestionClick: (String) -> Unit) {
    val suggestions = listOf("近くのカフェ", "今日のニュース", "天気予報", "おすすめレシピ")

    FlowRow(
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        modifier = Modifier.padding(16.dp)
    ) {
        suggestions.forEach { suggestion ->
            SuggestionChip(
                onClick = { onSuggestionClick(suggestion) },
                label = { Text(suggestion) },
                icon = { Icon(Icons.Default.Search, null, Modifier.size(18.dp)) }
            )
        }
    }
}
```

---

## 単一選択チップ

```kotlin
@Composable
fun SingleSelectChips() {
    val options = listOf("日", "週", "月", "年")
    var selected by remember { mutableStateOf("週") }

    Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
        options.forEach { option ->
            FilterChip(
                selected = option == selected,
                onClick = { selected = option },
                label = { Text(option) }
            )
        }
    }
}
```

---

## まとめ

- `FilterChip`で複数選択フィルター
- `InputChip`で削除可能なタグ表示
- `SuggestionChip`で提案候補
- `FlowRow`で折り返しレイアウト
- `selected`状態でチェックアイコン表示
- `trailingIcon`で削除ボタン追加

---

8種類のAndroidアプリテンプレート（Material3 Chip実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [検索/フィルターUI](https://zenn.dev/myougatheaxo/articles/android-compose-search-filter-2026)
- [Material3アイコン](https://zenn.dev/myougatheaxo/articles/android-compose-material3-icons-2026)
- [フォームバリデーション](https://zenn.dev/myougatheaxo/articles/android-compose-text-field-validation-2026)
