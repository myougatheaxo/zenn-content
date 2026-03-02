---
title: "Compose IconButton完全ガイド — FilledIconButton/OutlinedIconButton/トグル/バッジ付き"
emoji: "🔲"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**Compose IconButton**（IconButton、FilledIconButton、OutlinedIconButton、IconToggleButton、バッジ付き）を解説します。

---

## 基本IconButton

```kotlin
@Composable
fun IconButtonDemo() {
    Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
        // 標準
        IconButton(onClick = { /* 戻る */ }) {
            Icon(Icons.Default.ArrowBack, "戻る")
        }

        // Filled
        FilledIconButton(onClick = { /* 追加 */ }) {
            Icon(Icons.Default.Add, "追加")
        }

        // Filled Tonal
        FilledTonalIconButton(onClick = { /* 編集 */ }) {
            Icon(Icons.Default.Edit, "編集")
        }

        // Outlined
        OutlinedIconButton(onClick = { /* 共有 */ }) {
            Icon(Icons.Default.Share, "共有")
        }
    }
}
```

---

## IconToggleButton

```kotlin
@Composable
fun FavoriteToggle() {
    var isFavorite by remember { mutableStateOf(false) }

    IconToggleButton(checked = isFavorite, onCheckedChange = { isFavorite = it }) {
        Icon(
            imageVector = if (isFavorite) Icons.Filled.Favorite else Icons.Outlined.FavoriteBorder,
            contentDescription = "お気に入り",
            tint = if (isFavorite) Color.Red else MaterialTheme.colorScheme.onSurfaceVariant
        )
    }
}

// FilledIconToggleButton
@Composable
fun BookmarkToggle() {
    var isBookmarked by remember { mutableStateOf(false) }

    FilledIconToggleButton(checked = isBookmarked, onCheckedChange = { isBookmarked = it }) {
        Icon(
            imageVector = if (isBookmarked) Icons.Filled.Bookmark else Icons.Outlined.BookmarkBorder,
            contentDescription = "ブックマーク"
        )
    }
}
```

---

## バッジ付きIconButton

```kotlin
@Composable
fun BadgedIconButtonDemo() {
    var notificationCount by remember { mutableIntStateOf(3) }

    IconButton(onClick = { notificationCount = 0 }) {
        BadgedBox(badge = {
            if (notificationCount > 0) {
                Badge { Text("$notificationCount") }
            }
        }) {
            Icon(Icons.Default.Notifications, "通知")
        }
    }
}

// ツールバー例
@Composable
fun ToolbarDemo() {
    TopAppBar(
        title = { Text("マイアプリ") },
        navigationIcon = {
            IconButton(onClick = { /* メニュー */ }) {
                Icon(Icons.Default.Menu, "メニュー")
            }
        },
        actions = {
            IconButton(onClick = { /* 検索 */ }) { Icon(Icons.Default.Search, "検索") }
            IconButton(onClick = { /* 設定 */ }) { Icon(Icons.Default.Settings, "設定") }
        }
    )
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `IconButton` | 標準アイコンボタン |
| `FilledIconButton` | 塗りつぶしアイコンボタン |
| `OutlinedIconButton` | 枠線アイコンボタン |
| `IconToggleButton` | トグル式アイコンボタン |

- M3では4種類のIconButton（Standard/Filled/Tonal/Outlined）
- `IconToggleButton`でお気に入り等のON/OFF切替
- `BadgedBox`と組み合わせて通知バッジ表示
- `TopAppBar`の`actions`にIconButtonを配置

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose TopAppBar](https://zenn.dev/myougatheaxo/articles/android-compose-compose-top-app-bar-2026)
- [Compose Badge](https://zenn.dev/myougatheaxo/articles/android-compose-compose-badge-2026)
- [Compose MaterialIcons](https://zenn.dev/myougatheaxo/articles/android-compose-compose-material-icons-2026)
