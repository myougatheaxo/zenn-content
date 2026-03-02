---
title: "Badge完全ガイド — BadgedBox/数値バッジ/ドットバッジ/カスタム"
emoji: "🔴"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**Badge**（BadgedBox、数値バッジ、ドットバッジ、カスタムバッジ）を解説します。

---

## BadgedBox

```kotlin
@Composable
fun BadgeExamples() {
    Row(
        Modifier.padding(16.dp),
        horizontalArrangement = Arrangement.spacedBy(32.dp)
    ) {
        // 数値バッジ
        BadgedBox(badge = { Badge { Text("3") } }) {
            Icon(Icons.Default.Email, "メール")
        }

        // ドットバッジ
        BadgedBox(badge = { Badge() }) {
            Icon(Icons.Default.Notifications, "通知")
        }

        // 大きい数値
        BadgedBox(badge = { Badge { Text("99+") } }) {
            Icon(Icons.Default.ShoppingCart, "カート")
        }
    }
}
```

---

## NavigationBarバッジ

```kotlin
@Composable
fun NavBarWithBadge() {
    var selected by remember { mutableIntStateOf(0) }
    val badges = listOf(0, 5, 0) // 未読数

    NavigationBar {
        listOf(
            Triple("ホーム", Icons.Default.Home, Icons.Outlined.Home),
            Triple("通知", Icons.Default.Notifications, Icons.Outlined.Notifications),
            Triple("設定", Icons.Default.Settings, Icons.Outlined.Settings)
        ).forEachIndexed { index, (label, filled, outlined) ->
            NavigationBarItem(
                selected = selected == index,
                onClick = { selected = index },
                icon = {
                    BadgedBox(
                        badge = {
                            if (badges[index] > 0) Badge { Text("${badges[index]}") }
                        }
                    ) {
                        Icon(if (selected == index) filled else outlined, label)
                    }
                },
                label = { Text(label) }
            )
        }
    }
}
```

---

## カスタムバッジ

```kotlin
@Composable
fun CustomBadge(count: Int, content: @Composable () -> Unit) {
    Box {
        content()
        if (count > 0) {
            Box(
                modifier = Modifier
                    .align(Alignment.TopEnd)
                    .offset(x = 6.dp, y = (-6).dp)
                    .size(20.dp)
                    .background(Color.Red, CircleShape),
                contentAlignment = Alignment.Center
            ) {
                Text(
                    if (count > 99) "99+" else "$count",
                    color = Color.White,
                    fontSize = 10.sp,
                    fontWeight = FontWeight.Bold
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
| `BadgedBox` | バッジ付きアイコン |
| `Badge` | Material3バッジ |
| `Badge { Text() }` | 数値バッジ |
| `Badge()` | ドットバッジ |

- `BadgedBox`でアイコンにバッジ付与
- `Badge { Text("数値") }`で未読数表示
- `Badge()`で新着ドット表示
- `NavigationBar`+`BadgedBox`でナビバーバッジ

---

8種類のAndroidアプリテンプレート（Material3 UI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Bottom Navigation](https://zenn.dev/myougatheaxo/articles/android-compose-bottom-navigation-2026)
- [Material Icons](https://zenn.dev/myougatheaxo/articles/android-compose-compose-material-icons-2026)
- [TabLayout](https://zenn.dev/myougatheaxo/articles/android-compose-compose-tab-layout-2026)
