---
title: "Card完全ガイド — Card/ElevatedCard/OutlinedCard/カスタムCard"
emoji: "🃏"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**Card**（Card、ElevatedCard、OutlinedCard、クリック可能Card、カスタムCard）を解説します。

---

## Cardの種類

```kotlin
@Composable
fun CardVariants() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        // Filled Card（デフォルト）
        Card(Modifier.fillMaxWidth()) {
            Text("Filled Card", Modifier.padding(16.dp))
        }

        // Elevated Card
        ElevatedCard(
            modifier = Modifier.fillMaxWidth(),
            elevation = CardDefaults.elevatedCardElevation(defaultElevation = 6.dp)
        ) {
            Text("Elevated Card", Modifier.padding(16.dp))
        }

        // Outlined Card
        OutlinedCard(Modifier.fillMaxWidth()) {
            Text("Outlined Card", Modifier.padding(16.dp))
        }
    }
}
```

---

## クリック可能Card

```kotlin
@Composable
fun ClickableCardExample(item: Item, onClick: () -> Unit) {
    ElevatedCard(
        onClick = onClick,
        modifier = Modifier.fillMaxWidth()
    ) {
        Column {
            AsyncImage(
                model = item.imageUrl,
                contentDescription = null,
                modifier = Modifier.fillMaxWidth().height(180.dp),
                contentScale = ContentScale.Crop
            )
            Column(Modifier.padding(16.dp)) {
                Text(item.title, style = MaterialTheme.typography.titleMedium)
                Spacer(Modifier.height(4.dp))
                Text(item.description, maxLines = 2, overflow = TextOverflow.Ellipsis)
                Spacer(Modifier.height(8.dp))
                Row {
                    AssistChip(onClick = {}, label = { Text(item.category) })
                    Spacer(Modifier.weight(1f))
                    Text(item.date, fontSize = 12.sp, color = MaterialTheme.colorScheme.outline)
                }
            }
        }
    }
}
```

---

## カスタムCardカラー

```kotlin
@Composable
fun CustomColorCards() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(8.dp)) {
        Card(
            colors = CardDefaults.cardColors(
                containerColor = MaterialTheme.colorScheme.primaryContainer,
                contentColor = MaterialTheme.colorScheme.onPrimaryContainer
            )
        ) { Text("Primary Container", Modifier.padding(16.dp)) }

        Card(
            colors = CardDefaults.cardColors(
                containerColor = MaterialTheme.colorScheme.secondaryContainer
            )
        ) { Text("Secondary Container", Modifier.padding(16.dp)) }

        Card(
            colors = CardDefaults.cardColors(
                containerColor = MaterialTheme.colorScheme.tertiaryContainer
            )
        ) { Text("Tertiary Container", Modifier.padding(16.dp)) }
    }
}
```

---

## まとめ

| Card | スタイル |
|------|---------|
| `Card` | Filled（塗りつぶし） |
| `ElevatedCard` | 影付き |
| `OutlinedCard` | 枠線 |
| `onClick` | クリック対応 |

- 3種類のCardをコンテンツに応じて使い分け
- `ElevatedCard(onClick)`でクリック可能カード
- `CardDefaults.cardColors`でカスタムカラー
- 画像+テキスト+チップの複合Cardレイアウト

---

8種類のAndroidアプリテンプレート（Material3 UI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Surface](https://zenn.dev/myougatheaxo/articles/android-compose-compose-surface-2026)
- [LazyGrid](https://zenn.dev/myougatheaxo/articles/android-compose-compose-lazy-grid-2026)
- [画像読み込み](https://zenn.dev/myougatheaxo/articles/android-compose-compose-image-loading-2026)
