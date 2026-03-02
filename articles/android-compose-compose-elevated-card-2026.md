---
title: "Compose ElevatedCard完全ガイド — 画像カード/プロフィールカード/ダッシュボード"
emoji: "📇"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**Compose ElevatedCard**（画像カード、プロフィールカード、ダッシュボードカード、グリッドカード）を解説します。

---

## 画像カード

```kotlin
@Composable
fun ImageCardDemo() {
    ElevatedCard(
        modifier = Modifier.fillMaxWidth(),
        elevation = CardDefaults.elevatedCardElevation(defaultElevation = 4.dp)
    ) {
        AsyncImage(
            model = "https://example.com/image.jpg",
            contentDescription = null,
            modifier = Modifier.fillMaxWidth().height(200.dp),
            contentScale = ContentScale.Crop
        )
        Column(Modifier.padding(16.dp)) {
            Text("記事タイトル", style = MaterialTheme.typography.titleMedium)
            Text("2026年3月2日", style = MaterialTheme.typography.bodySmall,
                color = MaterialTheme.colorScheme.onSurfaceVariant)
            Spacer(Modifier.height(8.dp))
            Text("記事の概要テキストがここに表示されます...",
                style = MaterialTheme.typography.bodyMedium, maxLines = 2,
                overflow = TextOverflow.Ellipsis)
        }
    }
}
```

---

## プロフィールカード

```kotlin
@Composable
fun ProfileCard(name: String, role: String, avatarUrl: String) {
    ElevatedCard(
        Modifier.fillMaxWidth(),
        elevation = CardDefaults.elevatedCardElevation(defaultElevation = 2.dp)
    ) {
        Row(Modifier.padding(16.dp), verticalAlignment = Alignment.CenterVertically) {
            AsyncImage(
                model = avatarUrl, contentDescription = "アバター",
                modifier = Modifier.size(64.dp).clip(CircleShape),
                contentScale = ContentScale.Crop
            )
            Spacer(Modifier.width(16.dp))
            Column(Modifier.weight(1f)) {
                Text(name, style = MaterialTheme.typography.titleMedium)
                Text(role, style = MaterialTheme.typography.bodyMedium,
                    color = MaterialTheme.colorScheme.onSurfaceVariant)
            }
            FilledTonalButton(onClick = {}) { Text("フォロー") }
        }
    }
}
```

---

## ダッシュボードカード

```kotlin
@Composable
fun DashboardGrid() {
    data class Stat(val label: String, val value: String, val icon: ImageVector, val color: Color)
    val stats = listOf(
        Stat("売上", "¥123,456", Icons.Default.AttachMoney, Color(0xFF4CAF50)),
        Stat("ユーザー", "1,234", Icons.Default.People, Color(0xFF2196F3)),
        Stat("注文", "567", Icons.Default.ShoppingCart, Color(0xFFFF9800)),
        Stat("評価", "4.8", Icons.Default.Star, Color(0xFFF44336))
    )

    LazyVerticalGrid(columns = GridCells.Fixed(2), Modifier.padding(16.dp),
        horizontalArrangement = Arrangement.spacedBy(12.dp),
        verticalArrangement = Arrangement.spacedBy(12.dp)) {
        items(stats) { stat ->
            ElevatedCard {
                Column(Modifier.padding(16.dp)) {
                    Icon(stat.icon, null, tint = stat.color, modifier = Modifier.size(32.dp))
                    Spacer(Modifier.height(8.dp))
                    Text(stat.value, style = MaterialTheme.typography.headlineSmall)
                    Text(stat.label, style = MaterialTheme.typography.bodySmall,
                        color = MaterialTheme.colorScheme.onSurfaceVariant)
                }
            }
        }
    }
}
```

---

## まとめ

| パターン | 用途 |
|----------|------|
| 画像カード | 記事/商品一覧 |
| プロフィールカード | ユーザー情報 |
| ダッシュボード | 統計/KPI表示 |
| グリッドカード | タイル表示 |

- `ElevatedCard`は影で浮遊感を表現
- `CardDefaults.elevatedCardElevation()`でelevation調整
- `AsyncImage`と組み合わせて画像カード
- `LazyVerticalGrid`でダッシュボード風グリッド

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Card](https://zenn.dev/myougatheaxo/articles/android-compose-compose-outlined-card-2026)
- [Compose LazyVerticalGrid](https://zenn.dev/myougatheaxo/articles/android-compose-compose-lazy-vertical-grid-2026)
- [Compose Coil](https://zenn.dev/myougatheaxo/articles/android-compose-compose-coil-2026)
