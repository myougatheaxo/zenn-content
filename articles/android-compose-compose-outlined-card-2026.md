---
title: "Compose Card完全ガイド — Card/ElevatedCard/OutlinedCard/使い分け"
emoji: "🃏"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**Compose Card**（Card、ElevatedCard、OutlinedCard、カード使い分け、カードレイアウトパターン）を解説します。

---

## Card3種類

```kotlin
@Composable
fun CardVariantsDemo() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(16.dp)) {
        // Filled Card（デフォルト）
        Card(Modifier.fillMaxWidth()) {
            Text("Filled Card", Modifier.padding(16.dp))
        }

        // Elevated Card（影付き）
        ElevatedCard(Modifier.fillMaxWidth(), elevation = CardDefaults.elevatedCardElevation(defaultElevation = 4.dp)) {
            Text("Elevated Card", Modifier.padding(16.dp))
        }

        // Outlined Card（枠線）
        OutlinedCard(Modifier.fillMaxWidth()) {
            Text("Outlined Card", Modifier.padding(16.dp))
        }
    }
}
```

---

## クリック可能カード

```kotlin
@Composable
fun ClickableCardDemo() {
    var expanded by remember { mutableStateOf(false) }

    ElevatedCard(
        onClick = { expanded = !expanded },
        modifier = Modifier.fillMaxWidth()
    ) {
        Column(Modifier.padding(16.dp)) {
            Row(Modifier.fillMaxWidth(), horizontalArrangement = Arrangement.SpaceBetween,
                verticalAlignment = Alignment.CenterVertically) {
                Text("お知らせ", style = MaterialTheme.typography.titleMedium)
                Icon(
                    if (expanded) Icons.Default.ExpandLess else Icons.Default.ExpandMore,
                    "展開"
                )
            }
            AnimatedVisibility(visible = expanded) {
                Text("新機能が追加されました。詳細はこちらをご確認ください。",
                    Modifier.padding(top = 8.dp), style = MaterialTheme.typography.bodyMedium)
            }
        }
    }
}
```

---

## カードリスト

```kotlin
@Composable
fun CardListDemo() {
    data class Item(val title: String, val description: String, val icon: ImageVector)
    val items = listOf(
        Item("設定", "アプリの設定を変更", Icons.Default.Settings),
        Item("プロフィール", "ユーザー情報を編集", Icons.Default.Person),
        Item("通知", "通知設定を管理", Icons.Default.Notifications)
    )

    LazyColumn(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(8.dp)) {
        items(items) { item ->
            OutlinedCard(onClick = {}, Modifier.fillMaxWidth()) {
                ListItem(
                    headlineContent = { Text(item.title) },
                    supportingContent = { Text(item.description) },
                    leadingContent = {
                        Icon(item.icon, null, tint = MaterialTheme.colorScheme.primary)
                    },
                    trailingContent = {
                        Icon(Icons.Default.ChevronRight, "詳細")
                    }
                )
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `Card` | 塗りつぶしカード |
| `ElevatedCard` | 影付きカード |
| `OutlinedCard` | 枠線カード |
| `CardDefaults` | カラー/エレベーション設定 |

- M3では3種類のCardを視覚的重要度で使い分け
- `onClick`パラメータでクリック可能カード
- `AnimatedVisibility`と組み合わせて展開カード
- `ListItem`をカード内に配置してリストカードを実現

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose ListItem](https://zenn.dev/myougatheaxo/articles/android-compose-compose-list-item-2026)
- [Compose LazyColumn](https://zenn.dev/myougatheaxo/articles/android-compose-compose-lazy-column-2026)
- [Compose Animation](https://zenn.dev/myougatheaxo/articles/android-compose-compose-animation-2026)
