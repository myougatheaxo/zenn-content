---
title: "FlowRow/FlowColumn完全ガイド — タグクラウド/折り返しレイアウト/チップ"
emoji: "🔀"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "layout"]
published: true
---

## この記事で学べること

**FlowRow/FlowColumn**（タグクラウド、折り返しレイアウト、フィルターチップ、動的配置）を解説します。

---

## FlowRow基本

```kotlin
@OptIn(ExperimentalLayoutApi::class)
@Composable
fun TagCloud(tags: List<String>, selectedTags: Set<String>, onToggle: (String) -> Unit) {
    FlowRow(
        modifier = Modifier.fillMaxWidth().padding(16.dp),
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        tags.forEach { tag ->
            FilterChip(
                selected = tag in selectedTags,
                onClick = { onToggle(tag) },
                label = { Text(tag) },
                leadingIcon = if (tag in selectedTags) {
                    { Icon(Icons.Default.Check, null, Modifier.size(16.dp)) }
                } else null
            )
        }
    }
}
```

---

## FlowColumn

```kotlin
@OptIn(ExperimentalLayoutApi::class)
@Composable
fun FlowColumnExample(items: List<String>) {
    FlowColumn(
        modifier = Modifier.fillMaxSize().padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp),
        horizontalArrangement = Arrangement.spacedBy(16.dp),
        maxItemsInEachColumn = 5
    ) {
        items.forEach { item ->
            AssistChip(onClick = {}, label = { Text(item) })
        }
    }
}
```

---

## maxItemsInEachRow

```kotlin
@OptIn(ExperimentalLayoutApi::class)
@Composable
fun GridLikeFlowRow(items: List<Item>) {
    FlowRow(
        modifier = Modifier.fillMaxWidth().padding(16.dp),
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp),
        maxItemsInEachRow = 3
    ) {
        items.forEach { item ->
            Card(
                Modifier.weight(1f).aspectRatio(1f)
            ) {
                Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                    Text(item.name)
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
| `FlowRow` | 横方向折り返し |
| `FlowColumn` | 縦方向折り返し |
| `maxItemsInEachRow` | 行あたり最大数 |
| `weight` | 均等幅 |

- `FlowRow`でタグクラウド/フィルターチップ
- `FlowColumn`で縦方向の折り返し配置
- `maxItemsInEachRow`でグリッド的レイアウト
- `Arrangement.spacedBy`でアイテム間隔を指定

---

8種類のAndroidアプリテンプレート（レイアウト最適化済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [カスタムLayout](https://zenn.dev/myougatheaxo/articles/android-compose-compose-layout-modifier-2026)
- [LazyGrid](https://zenn.dev/myougatheaxo/articles/android-compose-compose-lazy-grid-2026)
- [ConstraintLayout](https://zenn.dev/myougatheaxo/articles/android-compose-constraint-layout-2026)
