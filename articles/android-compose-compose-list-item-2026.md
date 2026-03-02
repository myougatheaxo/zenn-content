---
title: "Compose ListItem完全ガイド — M3 ListItem/アイコン/スワイプ/セクション分け"
emoji: "📋"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**Compose ListItem**（M3 ListItem、leadingContent/trailingContent、セクション分け、スワイプ対応）を解説します。

---

## 基本ListItem

```kotlin
@Composable
fun ListItemDemo() {
    Column {
        // 1行
        ListItem(headlineContent = { Text("Wi-Fi") })
        HorizontalDivider()

        // 2行
        ListItem(
            headlineContent = { Text("Bluetooth") },
            supportingContent = { Text("接続済み: AirPods") }
        )
        HorizontalDivider()

        // 3行
        ListItem(
            headlineContent = { Text("モバイルデータ") },
            supportingContent = { Text("今月の使用量: 3.2GB\n制限: 10GB") },
            overlineContent = { Text("ネットワーク") }
        )
    }
}
```

---

## アイコン+トレイリング

```kotlin
@Composable
fun RichListItemDemo() {
    Column {
        ListItem(
            headlineContent = { Text("ダークモード") },
            supportingContent = { Text("外観を切り替え") },
            leadingContent = { Icon(Icons.Default.DarkMode, null) },
            trailingContent = {
                var checked by remember { mutableStateOf(false) }
                Switch(checked = checked, onCheckedChange = { checked = it })
            }
        )
        HorizontalDivider()
        ListItem(
            headlineContent = { Text("ストレージ") },
            supportingContent = { Text("23.4 GB / 64 GB 使用中") },
            leadingContent = {
                Icon(Icons.Default.Storage, null,
                    tint = MaterialTheme.colorScheme.primary)
            },
            trailingContent = { Text("37%") }
        )
        HorizontalDivider()
        ListItem(
            headlineContent = { Text("バージョン") },
            leadingContent = { Icon(Icons.Default.Info, null) },
            trailingContent = { Text("2.1.0", color = MaterialTheme.colorScheme.onSurfaceVariant) }
        )
    }
}
```

---

## セクション分けリスト

```kotlin
@Composable
fun SectionedListDemo() {
    LazyColumn {
        // セクション: 一般
        item {
            Text("一般", Modifier.padding(horizontal = 16.dp, vertical = 8.dp),
                style = MaterialTheme.typography.labelLarge,
                color = MaterialTheme.colorScheme.primary)
        }
        items(listOf("言語" to Icons.Default.Language, "テーマ" to Icons.Default.Palette)) { (title, icon) ->
            ListItem(
                headlineContent = { Text(title) },
                leadingContent = { Icon(icon, null) },
                trailingContent = { Icon(Icons.Default.ChevronRight, null) },
                modifier = Modifier.clickable { }
            )
        }

        item { HorizontalDivider(Modifier.padding(vertical = 8.dp)) }

        // セクション: アカウント
        item {
            Text("アカウント", Modifier.padding(horizontal = 16.dp, vertical = 8.dp),
                style = MaterialTheme.typography.labelLarge,
                color = MaterialTheme.colorScheme.primary)
        }
        items(listOf("プロフィール" to Icons.Default.Person, "セキュリティ" to Icons.Default.Lock)) { (title, icon) ->
            ListItem(
                headlineContent = { Text(title) },
                leadingContent = { Icon(icon, null) },
                trailingContent = { Icon(Icons.Default.ChevronRight, null) },
                modifier = Modifier.clickable { }
            )
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `ListItem` | M3リストアイテム |
| `headlineContent` | メインテキスト |
| `supportingContent` | サブテキスト |
| `leadingContent` | 左側アイコン/画像 |
| `trailingContent` | 右側ウィジェット |

- M3 `ListItem`で統一的なリスト表示
- `leadingContent`にアイコン、`trailingContent`にSwitch/Chevron
- `overlineContent`で3行構成
- セクションヘッダー+`HorizontalDivider`でグループ化

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Card](https://zenn.dev/myougatheaxo/articles/android-compose-compose-outlined-card-2026)
- [Compose LazyColumn](https://zenn.dev/myougatheaxo/articles/android-compose-compose-lazy-column-2026)
- [Compose SwipeToDismiss](https://zenn.dev/myougatheaxo/articles/android-compose-compose-swipe-to-dismiss-2026)
