---
title: "BadgedBox/通知バッジ完全ガイド — アイコンバッジ/カウント表示/アニメーション"
emoji: "🔴"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**BadgedBox/通知バッジ**（Material3 BadgedBox、カウント表示、ドットバッジ、アニメーション付きバッジ）を解説します。

---

## BadgedBox基本

```kotlin
@Composable
fun NotificationIcon(count: Int) {
    BadgedBox(
        badge = {
            if (count > 0) {
                Badge(
                    containerColor = MaterialTheme.colorScheme.error
                ) {
                    Text(
                        text = if (count > 99) "99+" else count.toString(),
                        style = MaterialTheme.typography.labelSmall
                    )
                }
            }
        }
    ) {
        Icon(Icons.Default.Notifications, "通知")
    }
}

// ドットバッジ（数字なし）
@Composable
fun DotBadgeIcon(hasNew: Boolean) {
    BadgedBox(
        badge = {
            if (hasNew) Badge()  // 数字なし → ドットのみ
        }
    ) {
        Icon(Icons.Default.Email, "メール")
    }
}
```

---

## BottomNavigation + バッジ

```kotlin
@Composable
fun BottomNavWithBadges(
    items: List<NavItem>,
    badgeCounts: Map<Int, Int>,
    selectedIndex: Int,
    onSelect: (Int) -> Unit
) {
    NavigationBar {
        items.forEachIndexed { index, item ->
            NavigationBarItem(
                selected = selectedIndex == index,
                onClick = { onSelect(index) },
                icon = {
                    BadgedBox(
                        badge = {
                            val count = badgeCounts[index] ?: 0
                            if (count > 0) {
                                Badge { Text(count.toString()) }
                            }
                        }
                    ) {
                        Icon(item.icon, item.label)
                    }
                },
                label = { Text(item.label) }
            )
        }
    }
}
```

---

## アニメーション付きバッジ

```kotlin
@Composable
fun AnimatedBadge(count: Int) {
    val scale by animateFloatAsState(
        targetValue = if (count > 0) 1f else 0f,
        animationSpec = spring(dampingRatio = Spring.DampingRatioMediumBouncy),
        label = "badgeScale"
    )

    BadgedBox(
        badge = {
            if (count > 0) {
                Badge(
                    modifier = Modifier.graphicsLayer {
                        scaleX = scale; scaleY = scale
                    }
                ) {
                    val animatedCount by animateIntAsState(
                        targetValue = count, label = "count"
                    )
                    Text(animatedCount.toString())
                }
            }
        }
    ) {
        Icon(Icons.Default.ShoppingCart, "カート")
    }
}
```

---

## カスタムバッジ

```kotlin
@Composable
fun CustomBadge(
    text: String,
    color: Color = MaterialTheme.colorScheme.error,
    modifier: Modifier = Modifier
) {
    Box(
        modifier
            .background(color, RoundedCornerShape(10.dp))
            .padding(horizontal = 6.dp, vertical = 2.dp),
        contentAlignment = Alignment.Center
    ) {
        Text(
            text = text,
            color = Color.White,
            style = MaterialTheme.typography.labelSmall,
            fontSize = 10.sp,
            fontWeight = FontWeight.Bold
        )
    }
}

// 使用: ステータスバッジ
@Composable
fun StatusItem(title: String, status: String) {
    Row(verticalAlignment = Alignment.CenterVertically) {
        Text(title, Modifier.weight(1f))
        CustomBadge(
            text = status,
            color = when (status) {
                "NEW" -> Color(0xFF4CAF50)
                "SALE" -> Color(0xFFF44336)
                else -> Color.Gray
            }
        )
    }
}
```

---

## まとめ

| バッジ種類 | 用途 |
|-----------|------|
| `Badge{}` | ドットバッジ |
| `Badge{Text()}` | カウントバッジ |
| カスタム | ステータス表示 |
| アニメーション | 注目効果 |

- `BadgedBox`でアイコンにバッジを重ねる
- `Badge()`で数字付き/ドットのみを切替
- `animateFloatAsState`でバッジ出現アニメーション
- NavigationBarItemと組み合わせてタブバッジ

---

8種類のAndroidアプリテンプレート（バッジUI実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Material3コンポーネント](https://zenn.dev/myougatheaxo/articles/android-compose-material3-components-2026)
- [通知](https://zenn.dev/myougatheaxo/articles/android-compose-notification-advanced-2026)
- [Navigation](https://zenn.dev/myougatheaxo/articles/android-compose-type-safe-navigation-2026)
