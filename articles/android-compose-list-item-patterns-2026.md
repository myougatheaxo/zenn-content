---
title: "リストアイテムパターン集 — ListItem/Card/SwipeでプロなUI"
emoji: "📋"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "list"]
published: true
---

## この記事で学べること

Composeの**リストアイテム**の実装パターン（ListItem、Card、スワイプアクション）を解説します。

---

## 基本のListItem

```kotlin
@Composable
fun BasicListItems() {
    LazyColumn {
        // 1行ListItem
        item {
            ListItem(headlineContent = { Text("タイトル") })
        }

        // 2行ListItem
        item {
            ListItem(
                headlineContent = { Text("タイトル") },
                supportingContent = { Text("サブテキスト") }
            )
        }

        // 3行ListItem
        item {
            ListItem(
                headlineContent = { Text("タイトル") },
                supportingContent = { Text("説明文がここに入ります。長い場合は複数行表示されます。") },
                overlineContent = { Text("カテゴリ") }
            )
        }

        // アイコン・末尾要素付き
        item {
            ListItem(
                headlineContent = { Text("設定") },
                supportingContent = { Text("アプリの設定を変更") },
                leadingContent = {
                    Icon(Icons.Default.Settings, null, Modifier.size(40.dp))
                },
                trailingContent = {
                    Icon(Icons.Default.ChevronRight, null)
                }
            )
        }
    }
}
```

---

## アバター付きリスト

```kotlin
@Composable
fun UserListItem(user: User, onClick: () -> Unit) {
    ListItem(
        headlineContent = { Text(user.name) },
        supportingContent = { Text(user.email) },
        leadingContent = {
            AsyncImage(
                model = user.avatarUrl,
                contentDescription = null,
                modifier = Modifier.size(48.dp).clip(CircleShape),
                contentScale = ContentScale.Crop
            )
        },
        trailingContent = {
            Text(
                user.lastActive,
                style = MaterialTheme.typography.bodySmall,
                color = MaterialTheme.colorScheme.outline
            )
        },
        modifier = Modifier.clickable(onClick = onClick)
    )
}
```

---

## カード型リスト

```kotlin
@Composable
fun ProductCard(product: Product, onAddToCart: () -> Unit) {
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .padding(horizontal = 16.dp, vertical = 4.dp),
        elevation = CardDefaults.cardElevation(defaultElevation = 2.dp)
    ) {
        Row(Modifier.padding(12.dp)) {
            AsyncImage(
                model = product.imageUrl,
                contentDescription = null,
                modifier = Modifier
                    .size(80.dp)
                    .clip(RoundedCornerShape(8.dp)),
                contentScale = ContentScale.Crop
            )

            Spacer(Modifier.width(12.dp))

            Column(Modifier.weight(1f)) {
                Text(product.name, style = MaterialTheme.typography.titleMedium)
                Text(
                    product.description,
                    style = MaterialTheme.typography.bodySmall,
                    maxLines = 2,
                    overflow = TextOverflow.Ellipsis
                )
                Spacer(Modifier.height(8.dp))
                Row(verticalAlignment = Alignment.CenterVertically) {
                    Text(
                        "¥${product.price}",
                        style = MaterialTheme.typography.titleSmall,
                        color = MaterialTheme.colorScheme.primary
                    )
                    Spacer(Modifier.weight(1f))
                    FilledTonalButton(onClick = onAddToCart) {
                        Text("カートに追加")
                    }
                }
            }
        }
    }
}
```

---

## チェック選択リスト

```kotlin
@Composable
fun SelectableList(items: List<Item>) {
    var selectedIds by remember { mutableStateOf(setOf<String>()) }

    LazyColumn {
        items(items, key = { it.id }) { item ->
            ListItem(
                headlineContent = { Text(item.title) },
                leadingContent = {
                    Checkbox(
                        checked = item.id in selectedIds,
                        onCheckedChange = {
                            selectedIds = if (it) selectedIds + item.id
                                          else selectedIds - item.id
                        }
                    )
                },
                modifier = Modifier.clickable {
                    selectedIds = if (item.id in selectedIds) selectedIds - item.id
                                  else selectedIds + item.id
                }
            )
        }
    }

    // 選択数表示
    if (selectedIds.isNotEmpty()) {
        Text("${selectedIds.size}件選択中")
    }
}
```

---

## セクション付きリスト

```kotlin
@Composable
fun SectionedList(groupedItems: Map<String, List<Item>>) {
    LazyColumn {
        groupedItems.forEach { (section, items) ->
            stickyHeader {
                Text(
                    section,
                    Modifier
                        .fillMaxWidth()
                        .background(MaterialTheme.colorScheme.surfaceVariant)
                        .padding(horizontal = 16.dp, vertical = 8.dp),
                    style = MaterialTheme.typography.titleSmall,
                    color = MaterialTheme.colorScheme.primary
                )
            }
            items(items, key = { it.id }) { item ->
                ListItem(
                    headlineContent = { Text(item.title) },
                    supportingContent = { Text(item.subtitle) }
                )
            }
        }
    }
}
```

---

## まとめ

- `ListItem`でMaterial 3標準のリスト行
- `headlineContent`/`supportingContent`/`overlineContent`で最大3行
- `leadingContent`/`trailingContent`で左右アイコン
- `Card` + `Row`でリッチなカード型アイテム
- `Checkbox` + `selectedIds`で複数選択
- `stickyHeader`でセクションヘッダー

---

8種類のAndroidアプリテンプレート（リストUI設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [LazyColumn完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-lazycolumn-guide-2026)
- [SwipeActionsガイド](https://zenn.dev/myougatheaxo/articles/android-compose-swipe-actions-2026)
- [画像読み込みガイド](https://zenn.dev/myougatheaxo/articles/android-compose-image-loading-2026)
