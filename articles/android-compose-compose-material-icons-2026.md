---
title: "Material Icons完全ガイド — アイコン一覧/Extended/カスタムアイコン/ベクタ"
emoji: "🎯"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**Material Icons**（Icons.Default、Icons.Outlined、Extended Icons、カスタムベクタアイコン）を解説します。

---

## Iconsの種類

```kotlin
@Composable
fun IconStyleShowcase() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        // Default (Filled)
        Row(verticalAlignment = Alignment.CenterVertically) {
            Icon(Icons.Default.Home, "ホーム")
            Spacer(Modifier.width(8.dp))
            Text("Icons.Default (Filled)")
        }

        // Outlined
        Row(verticalAlignment = Alignment.CenterVertically) {
            Icon(Icons.Outlined.Home, "ホーム")
            Spacer(Modifier.width(8.dp))
            Text("Icons.Outlined")
        }

        // Rounded
        Row(verticalAlignment = Alignment.CenterVertically) {
            Icon(Icons.Rounded.Home, "ホーム")
            Spacer(Modifier.width(8.dp))
            Text("Icons.Rounded")
        }

        // Sharp
        Row(verticalAlignment = Alignment.CenterVertically) {
            Icon(Icons.Sharp.Home, "ホーム")
            Spacer(Modifier.width(8.dp))
            Text("Icons.Sharp")
        }

        // TwoTone
        Row(verticalAlignment = Alignment.CenterVertically) {
            Icon(Icons.TwoTone.Home, "ホーム")
            Spacer(Modifier.width(8.dp))
            Text("Icons.TwoTone")
        }
    }
}
```

---

## Extended Icons

```kotlin
// build.gradle.kts
// implementation("androidx.compose.material:material-icons-extended")

@Composable
fun ExtendedIconsExample() {
    FlowRow(
        Modifier.padding(16.dp),
        horizontalArrangement = Arrangement.spacedBy(16.dp),
        verticalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        listOf(
            Icons.Default.AccountCircle to "アカウント",
            Icons.Default.ShoppingCart to "カート",
            Icons.Default.Notifications to "通知",
            Icons.Default.Favorite to "お気に入り",
            Icons.Default.Share to "共有",
            Icons.Default.Download to "DL"
        ).forEach { (icon, label) ->
            Column(horizontalAlignment = Alignment.CenterHorizontally) {
                Icon(icon, label, modifier = Modifier.size(32.dp))
                Text(label, fontSize = 10.sp)
            }
        }
    }
}
```

---

## カスタムベクタアイコン

```kotlin
@Composable
fun CustomVectorIcon() {
    // XMLベクタ
    Icon(
        painter = painterResource(R.drawable.ic_custom),
        contentDescription = "カスタム",
        modifier = Modifier.size(24.dp),
        tint = MaterialTheme.colorScheme.primary
    )
}

// アイコンボタン with Badge
@Composable
fun NotificationIcon(count: Int) {
    BadgedBox(
        badge = {
            if (count > 0) {
                Badge { Text(if (count > 99) "99+" else "$count") }
            }
        }
    ) {
        Icon(Icons.Default.Notifications, "通知")
    }
}
```

---

## まとめ

| スタイル | 依存関係 |
|---------|---------|
| `Icons.Default` | material3（標準） |
| `Icons.Outlined` | material3（標準） |
| Extended Icons | material-icons-extended |
| カスタム | `painterResource` |

- `Icons.Default`で基本的なMaterial Iconsを使用
- Extended Iconsで900+のアイコンを追加
- `painterResource`でカスタムSVG/XMLベクタ
- `tint`でアイコン色をテーマに合わせる

---

8種類のAndroidアプリテンプレート（Material3アイコン対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Typography](https://zenn.dev/myougatheaxo/articles/android-compose-compose-typography-2026)
- [Chip](https://zenn.dev/myougatheaxo/articles/android-compose-compose-chip-2026)
- [Bottom Navigation](https://zenn.dev/myougatheaxo/articles/android-compose-bottom-navigation-2026)
