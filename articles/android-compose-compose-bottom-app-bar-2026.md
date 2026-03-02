---
title: "BottomAppBar完全ガイド — BottomAppBar/FAB連携/アクション/スクロール非表示"
emoji: "📱"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**BottomAppBar**（BottomAppBar、FAB連携、アクションアイコン、スクロール非表示）を解説します。

---

## 基本BottomAppBar

```kotlin
@Composable
fun BottomAppBarExample() {
    Scaffold(
        bottomBar = {
            BottomAppBar(
                actions = {
                    IconButton(onClick = {}) {
                        Icon(Icons.Default.Check, "完了")
                    }
                    IconButton(onClick = {}) {
                        Icon(Icons.Default.Edit, "編集")
                    }
                    IconButton(onClick = {}) {
                        Icon(Icons.Default.Share, "共有")
                    }
                    IconButton(onClick = {}) {
                        Icon(Icons.Default.Delete, "削除")
                    }
                },
                floatingActionButton = {
                    FloatingActionButton(
                        onClick = {},
                        containerColor = BottomAppBarDefaults.bottomAppBarFabColor
                    ) {
                        Icon(Icons.Default.Add, "追加")
                    }
                }
            )
        }
    ) { padding ->
        LazyColumn(Modifier.padding(padding)) {
            items(30) { ListItem(headlineContent = { Text("Item $it") }) }
        }
    }
}
```

---

## スクロール非表示

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun ScrollableBottomBar() {
    val scrollBehavior = BottomAppBarDefaults.exitAlwaysScrollBehavior()

    Scaffold(
        modifier = Modifier.nestedScroll(scrollBehavior.nestedScrollConnection),
        bottomBar = {
            BottomAppBar(
                actions = {
                    IconButton(onClick = {}) { Icon(Icons.Default.Home, "ホーム") }
                    IconButton(onClick = {}) { Icon(Icons.Default.Search, "検索") }
                },
                floatingActionButton = {
                    FloatingActionButton(onClick = {}) {
                        Icon(Icons.Default.Add, "追加")
                    }
                },
                scrollBehavior = scrollBehavior
            )
        }
    ) { padding ->
        LazyColumn(Modifier.padding(padding)) {
            items(50) { ListItem(headlineContent = { Text("Item $it") }) }
        }
    }
}
```

---

## カスタムBottomAppBar

```kotlin
@Composable
fun CustomBottomBar(selectedItem: Int, onItemSelected: (Int) -> Unit) {
    BottomAppBar(
        containerColor = MaterialTheme.colorScheme.surface,
        tonalElevation = 3.dp
    ) {
        Row(
            Modifier.fillMaxWidth(),
            horizontalArrangement = Arrangement.SpaceEvenly
        ) {
            listOf(
                Icons.Default.Home to "ホーム",
                Icons.Default.Search to "検索",
                Icons.Default.Favorite to "お気に入り",
                Icons.Default.Person to "プロフィール"
            ).forEachIndexed { index, (icon, label) ->
                Column(
                    Modifier
                        .clickable { onItemSelected(index) }
                        .padding(8.dp),
                    horizontalAlignment = Alignment.CenterHorizontally
                ) {
                    Icon(
                        icon, label,
                        tint = if (selectedItem == index)
                            MaterialTheme.colorScheme.primary
                        else MaterialTheme.colorScheme.onSurfaceVariant
                    )
                    Text(label, fontSize = 10.sp)
                }
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `BottomAppBar` | 下部アプリバー |
| `actions` | アクションアイコン |
| `floatingActionButton` | FAB統合 |
| `exitAlwaysScrollBehavior` | スクロール非表示 |

- `BottomAppBar`+FABでMaterial3の下部バー
- `actions`で複数のアクションアイコン配置
- `exitAlwaysScrollBehavior`でスクロール時非表示
- カスタムレイアウトで独自の下部バー

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Scaffold](https://zenn.dev/myougatheaxo/articles/android-compose-compose-scaffold-2026)
- [Bottom Navigation](https://zenn.dev/myougatheaxo/articles/android-compose-bottom-navigation-2026)
- [NestedScroll](https://zenn.dev/myougatheaxo/articles/android-compose-compose-nested-scroll-2026)
