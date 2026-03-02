---
title: "展開リスト完全ガイド — アコーディオン/ExpandableCard/ツリービュー"
emoji: "📂"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**展開リスト**（アコーディオン、ExpandableCard、ツリービュー、アニメーション付き展開）を解説します。

---

## ExpandableCard

```kotlin
@Composable
fun ExpandableCard(title: String, content: @Composable () -> Unit) {
    var expanded by remember { mutableStateOf(false) }
    val rotationAngle by animateFloatAsState(if (expanded) 180f else 0f, label = "rotation")

    Card(
        modifier = Modifier.fillMaxWidth().padding(vertical = 4.dp),
        onClick = { expanded = !expanded }
    ) {
        Column(Modifier.padding(16.dp)) {
            Row(
                Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.SpaceBetween,
                verticalAlignment = Alignment.CenterVertically
            ) {
                Text(title, style = MaterialTheme.typography.titleMedium)
                Icon(
                    Icons.Default.ExpandMore, "展開",
                    modifier = Modifier.rotate(rotationAngle)
                )
            }

            AnimatedVisibility(
                visible = expanded,
                enter = expandVertically() + fadeIn(),
                exit = shrinkVertically() + fadeOut()
            ) {
                Column(Modifier.padding(top = 12.dp)) {
                    content()
                }
            }
        }
    }
}

// 使用例: FAQ
@Composable
fun FaqList(items: List<FaqItem>) {
    LazyColumn(Modifier.padding(16.dp)) {
        items(items) { faq ->
            ExpandableCard(title = faq.question) {
                Text(faq.answer, style = MaterialTheme.typography.bodyMedium)
            }
        }
    }
}
```

---

## アコーディオン（排他展開）

```kotlin
@Composable
fun AccordionList(sections: List<Section>) {
    var expandedIndex by remember { mutableIntStateOf(-1) }

    LazyColumn(Modifier.padding(16.dp)) {
        itemsIndexed(sections) { index, section ->
            Card(
                modifier = Modifier.fillMaxWidth().padding(vertical = 4.dp),
                onClick = { expandedIndex = if (expandedIndex == index) -1 else index }
            ) {
                Column(Modifier.padding(16.dp)) {
                    Row(
                        Modifier.fillMaxWidth(),
                        horizontalArrangement = Arrangement.SpaceBetween
                    ) {
                        Text(section.title, style = MaterialTheme.typography.titleMedium)
                        Icon(
                            if (expandedIndex == index) Icons.Default.ExpandLess else Icons.Default.ExpandMore,
                            "toggle"
                        )
                    }

                    AnimatedVisibility(visible = expandedIndex == index) {
                        Column(Modifier.padding(top = 8.dp)) {
                            section.items.forEach { item ->
                                ListItem(headlineContent = { Text(item) })
                            }
                        }
                    }
                }
            }
        }
    }
}
```

---

## ツリービュー

```kotlin
data class TreeNode(val label: String, val children: List<TreeNode> = emptyList())

@Composable
fun TreeView(nodes: List<TreeNode>, depth: Int = 0) {
    nodes.forEach { node ->
        var expanded by remember { mutableStateOf(false) }
        val hasChildren = node.children.isNotEmpty()

        Row(
            Modifier
                .fillMaxWidth()
                .clickable(enabled = hasChildren) { expanded = !expanded }
                .padding(start = (depth * 24).dp, top = 4.dp, bottom = 4.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            if (hasChildren) {
                Icon(
                    if (expanded) Icons.Default.ExpandMore else Icons.Default.ChevronRight,
                    null, Modifier.size(20.dp)
                )
            } else {
                Spacer(Modifier.size(20.dp))
            }
            Spacer(Modifier.width(4.dp))
            Text(node.label)
        }

        AnimatedVisibility(visible = expanded) {
            TreeView(nodes = node.children, depth = depth + 1)
        }
    }
}
```

---

## まとめ

| パターン | 動作 |
|---------|------|
| ExpandableCard | 個別展開/折りたたみ |
| アコーディオン | 排他展開（1つだけ） |
| ツリービュー | 階層的展開 |

- `AnimatedVisibility`で滑らかな展開アニメーション
- `expandVertically`+`fadeIn`で自然な表示
- アコーディオンは`expandedIndex`で排他制御
- ツリービューは再帰`@Composable`で実現

---

8種類のAndroidアプリテンプレート（リストUI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [LazyLayout](https://zenn.dev/myougatheaxo/articles/android-compose-lazy-layout-2026)
- [アニメーション](https://zenn.dev/myougatheaxo/articles/android-compose-animation-spring-2026)
- [Swipe Actions](https://zenn.dev/myougatheaxo/articles/android-compose-swipe-action-2026)
