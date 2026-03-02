---
title: "Badge・カウンター実装ガイド — Composeで通知バッジを表示"
emoji: "🔴"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

Material3の**Badge**と**BadgedBox**で通知バッジ・カウンターを実装する方法を解説します。

---

## 基本のBadge

```kotlin
@Composable
fun BadgeExample() {
    BadgedBox(
        badge = {
            Badge { Text("3") }
        }
    ) {
        Icon(Icons.Default.Notifications, "通知")
    }
}
```

---

## ドット型Badge

```kotlin
BadgedBox(
    badge = {
        Badge() // テキストなしでドット表示
    }
) {
    Icon(Icons.Default.Email, "メール")
}
```

---

## BottomNavigationでのBadge

```kotlin
@Composable
fun BottomNavWithBadge(unreadCount: Int) {
    NavigationBar {
        NavigationBarItem(
            icon = {
                BadgedBox(
                    badge = {
                        if (unreadCount > 0) {
                            Badge {
                                Text(
                                    if (unreadCount > 99) "99+" else "$unreadCount"
                                )
                            }
                        }
                    }
                ) {
                    Icon(Icons.Default.Email, null)
                }
            },
            label = { Text("メール") },
            selected = true,
            onClick = {}
        )

        NavigationBarItem(
            icon = {
                BadgedBox(badge = { Badge() }) {
                    Icon(Icons.Default.Chat, null)
                }
            },
            label = { Text("チャット") },
            selected = false,
            onClick = {}
        )

        NavigationBarItem(
            icon = { Icon(Icons.Default.Person, null) },
            label = { Text("プロフィール") },
            selected = false,
            onClick = {}
        )
    }
}
```

---

## カスタムBadge

```kotlin
@Composable
fun CustomBadge(count: Int) {
    Box {
        Icon(
            Icons.Default.ShoppingCart,
            "カート",
            modifier = Modifier.size(32.dp)
        )
        if (count > 0) {
            Box(
                modifier = Modifier
                    .align(Alignment.TopEnd)
                    .offset(x = 6.dp, y = (-6).dp)
                    .size(20.dp)
                    .clip(CircleShape)
                    .background(MaterialTheme.colorScheme.error),
                contentAlignment = Alignment.Center
            ) {
                Text(
                    "$count",
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

## アニメーション付きBadge

```kotlin
@Composable
fun AnimatedBadge(count: Int) {
    val animatedCount by animateIntAsState(
        targetValue = count,
        animationSpec = tween(300)
    )
    val scale by animateFloatAsState(
        targetValue = if (count > 0) 1f else 0f,
        animationSpec = spring(dampingRatio = Spring.DampingRatioMediumBouncy)
    )

    BadgedBox(
        badge = {
            if (count > 0) {
                Badge(
                    modifier = Modifier.graphicsLayer {
                        scaleX = scale
                        scaleY = scale
                    }
                ) {
                    Text("$animatedCount")
                }
            }
        }
    ) {
        Icon(Icons.Default.Notifications, "通知")
    }
}
```

---

## まとめ

- `BadgedBox` + `Badge`で通知バッジ
- テキストなし`Badge()`でドット表示
- `99+`表示で大きい数値に対応
- カスタムBadgeは`Box` + `offset`で配置
- `animateFloatAsState`でバウンスアニメーション

---

8種類のAndroidアプリテンプレート（バッジ通知UI設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [BottomNavigation完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-bottom-nav-2026)
- [TopAppBar完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-app-bar-2026)
- [Composeアニメーション入門](https://zenn.dev/myougatheaxo/articles/compose-animation-basics-2026)
