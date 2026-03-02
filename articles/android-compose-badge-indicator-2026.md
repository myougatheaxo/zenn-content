---
title: "バッジ/インジケーターガイド — 未読バッジ/ステータスドット/進捗表示"
emoji: "🔴"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

Composeでの**バッジ/インジケーター**（未読バッジ、ステータスドット、進捗表示）を解説します。

---

## Material3 Badge

```kotlin
@Composable
fun BadgeExamples() {
    Row(horizontalArrangement = Arrangement.spacedBy(24.dp)) {
        // 数値バッジ
        BadgedBox(badge = { Badge { Text("5") } }) {
            Icon(Icons.Default.Notifications, "通知")
        }

        // ドットバッジ
        BadgedBox(badge = { Badge() }) {
            Icon(Icons.Default.Mail, "メール")
        }

        // 大きい数値
        BadgedBox(badge = { Badge { Text("99+") } }) {
            Icon(Icons.Default.ShoppingCart, "カート")
        }
    }
}
```

---

## カスタムバッジ

```kotlin
@Composable
fun CustomBadge(
    count: Int,
    modifier: Modifier = Modifier,
    content: @Composable () -> Unit
) {
    Box(modifier) {
        content()
        if (count > 0) {
            Box(
                Modifier
                    .align(Alignment.TopEnd)
                    .offset(x = 6.dp, y = (-6).dp)
                    .size(if (count > 9) 20.dp else 16.dp)
                    .clip(CircleShape)
                    .background(MaterialTheme.colorScheme.error),
                contentAlignment = Alignment.Center
            ) {
                Text(
                    if (count > 99) "99+" else "$count",
                    color = MaterialTheme.colorScheme.onError,
                    style = MaterialTheme.typography.labelSmall,
                    fontSize = 10.sp
                )
            }
        }
    }
}
```

---

## ステータスインジケーター

```kotlin
enum class Status(val color: Color, val label: String) {
    ONLINE(Color(0xFF4CAF50), "オンライン"),
    AWAY(Color(0xFFFF9800), "離席中"),
    BUSY(Color(0xFFF44336), "取り込み中"),
    OFFLINE(Color.Gray, "オフライン")
}

@Composable
fun StatusIndicator(status: Status) {
    Row(verticalAlignment = Alignment.CenterVertically) {
        Box(
            Modifier
                .size(10.dp)
                .clip(CircleShape)
                .background(status.color)
        )
        Spacer(Modifier.width(8.dp))
        Text(status.label, style = MaterialTheme.typography.bodySmall)
    }
}

// アバター付きステータス
@Composable
fun AvatarWithStatus(imageUrl: String, status: Status) {
    Box {
        AsyncImage(
            model = imageUrl,
            contentDescription = null,
            modifier = Modifier.size(48.dp).clip(CircleShape),
            contentScale = ContentScale.Crop
        )
        Box(
            Modifier
                .align(Alignment.BottomEnd)
                .size(14.dp)
                .clip(CircleShape)
                .background(MaterialTheme.colorScheme.surface)
                .padding(2.dp)
                .clip(CircleShape)
                .background(status.color)
        )
    }
}
```

---

## プログレスインジケーター

```kotlin
@Composable
fun ProgressBadge(progress: Float, label: String) {
    Column(horizontalAlignment = Alignment.CenterHorizontally) {
        Box(contentAlignment = Alignment.Center) {
            CircularProgressIndicator(
                progress = { progress },
                modifier = Modifier.size(48.dp),
                strokeWidth = 4.dp,
                trackColor = MaterialTheme.colorScheme.surfaceVariant
            )
            Text(
                "${(progress * 100).toInt()}%",
                style = MaterialTheme.typography.labelSmall
            )
        }
        Spacer(Modifier.height(4.dp))
        Text(label, style = MaterialTheme.typography.bodySmall)
    }
}
```

---

## まとめ

- `BadgedBox` + `Badge`でMaterial3バッジ
- `Badge()`（引数なし）でドットバッジ
- `Badge { Text("5") }`で数値バッジ
- カスタムバッジは`Box` + `Alignment.TopEnd` + `offset`
- ステータスドットは`CircleShape` + 色で表現
- `CircularProgressIndicator`でプログレスバッジ

---

8種類のAndroidアプリテンプレート（バッジ/インジケーター設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [BottomNavigation+バッジガイド](https://zenn.dev/myougatheaxo/articles/android-compose-bottom-nav-badge-2026)
- [Material3コンポーネント一覧](https://zenn.dev/myougatheaxo/articles/compose-material3-components-2026)
- [プログレスインジケーターガイド](https://zenn.dev/myougatheaxo/articles/android-compose-progress-indicator-2026)
