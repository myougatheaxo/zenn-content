---
title: "TopAppBar完全ガイド — CenterAligned/Medium/Large/スクロール連携"
emoji: "🔝"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**TopAppBar**（CenterAlignedTopAppBar、MediumTopAppBar、LargeTopAppBar、スクロール連携）を解説します。

---

## TopAppBarの種類

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun TopAppBarShowcase() {
    var barType by remember { mutableIntStateOf(0) }

    Scaffold(
        topBar = {
            when (barType) {
                0 -> TopAppBar(
                    title = { Text("Small") },
                    navigationIcon = {
                        IconButton(onClick = {}) { Icon(Icons.Default.Menu, "メニュー") }
                    },
                    actions = {
                        IconButton(onClick = {}) { Icon(Icons.Default.Search, "検索") }
                        IconButton(onClick = {}) { Icon(Icons.Default.MoreVert, "その他") }
                    }
                )
                1 -> CenterAlignedTopAppBar(
                    title = { Text("Center Aligned") },
                    navigationIcon = {
                        IconButton(onClick = {}) { Icon(Icons.AutoMirrored.Filled.ArrowBack, "戻る") }
                    }
                )
                2 -> MediumTopAppBar(title = { Text("Medium") })
                3 -> LargeTopAppBar(title = { Text("Large") })
            }
        }
    ) { padding ->
        Column(Modifier.padding(padding).padding(16.dp)) {
            Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
                listOf("Small", "Center", "Medium", "Large").forEachIndexed { i, label ->
                    FilterChip(selected = barType == i, onClick = { barType = i }, label = { Text(label) })
                }
            }
        }
    }
}
```

---

## Collapsing TopAppBar

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun CollapsingTopBar() {
    val scrollBehavior = TopAppBarDefaults.exitUntilCollapsedScrollBehavior()

    Scaffold(
        modifier = Modifier.nestedScroll(scrollBehavior.nestedScrollConnection),
        topBar = {
            LargeTopAppBar(
                title = { Text("記事一覧") },
                navigationIcon = {
                    IconButton(onClick = {}) {
                        Icon(Icons.AutoMirrored.Filled.ArrowBack, "戻る")
                    }
                },
                actions = {
                    IconButton(onClick = {}) { Icon(Icons.Default.Search, "検索") }
                },
                scrollBehavior = scrollBehavior,
                colors = TopAppBarDefaults.largeTopAppBarColors(
                    scrolledContainerColor = MaterialTheme.colorScheme.surface
                )
            )
        }
    ) { padding ->
        LazyColumn(Modifier.padding(padding)) {
            items(50) { ListItem(headlineContent = { Text("記事 $it") }) }
        }
    }
}
```

---

## まとめ

| TopAppBar | 用途 |
|-----------|------|
| `TopAppBar` | 標準（小） |
| `CenterAlignedTopAppBar` | 中央揃え |
| `MediumTopAppBar` | 中サイズ |
| `LargeTopAppBar` | 大サイズ（Collapsing） |

- 4種類のTopAppBarをシーンに応じて使い分け
- `scrollBehavior`でスクロール連携
- `exitUntilCollapsedScrollBehavior`でCollapsing効果
- `navigationIcon`/`actions`でアイコン配置

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Scaffold](https://zenn.dev/myougatheaxo/articles/android-compose-compose-scaffold-2026)
- [NestedScroll](https://zenn.dev/myougatheaxo/articles/android-compose-compose-nested-scroll-2026)
- [BottomAppBar](https://zenn.dev/myougatheaxo/articles/android-compose-compose-bottom-app-bar-2026)
