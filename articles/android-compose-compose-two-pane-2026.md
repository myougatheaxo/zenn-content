---
title: "TwoPane完全ガイド — リスト詳細/マスター詳細/折りたたみ対応"
emoji: "📱"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "layout"]
published: true
---

## この記事で学べること

**TwoPane**（リスト詳細パターン、マスター/詳細レイアウト、Foldable対応）を解説します。

---

## リスト詳細パターン

```kotlin
@Composable
fun ListDetailScreen() {
    val windowInfo = currentWindowAdaptiveInfo()
    val isExpanded = windowInfo.windowSizeClass.windowWidthSizeClass == WindowWidthSizeClass.EXPANDED
    var selectedId by remember { mutableStateOf<Long?>(null) }
    val items = (1L..20L).map { Item(it, "Item $it", "詳細テキスト $it") }

    if (isExpanded) {
        // 横並びレイアウト
        Row(Modifier.fillMaxSize()) {
            ItemList(items, selectedId, { selectedId = it }, Modifier.weight(0.4f))
            VerticalDivider()
            if (selectedId != null) {
                ItemDetail(items.first { it.id == selectedId }, Modifier.weight(0.6f))
            } else {
                Box(Modifier.weight(0.6f), contentAlignment = Alignment.Center) {
                    Text("アイテムを選択してください", color = Color.Gray)
                }
            }
        }
    } else {
        // スタック型（Navigation）
        if (selectedId != null) {
            ItemDetail(items.first { it.id == selectedId }, Modifier.fillMaxSize())
            BackHandler { selectedId = null }
        } else {
            ItemList(items, selectedId, { selectedId = it }, Modifier.fillMaxSize())
        }
    }
}

@Composable
fun ItemList(items: List<Item>, selectedId: Long?, onSelect: (Long) -> Unit, modifier: Modifier) {
    LazyColumn(modifier) {
        items(items) { item ->
            ListItem(
                headlineContent = { Text(item.title) },
                modifier = Modifier.clickable { onSelect(item.id) }
                    .then(if (item.id == selectedId) Modifier.background(MaterialTheme.colorScheme.secondaryContainer) else Modifier)
            )
        }
    }
}

@Composable
fun ItemDetail(item: Item, modifier: Modifier) {
    Column(modifier.padding(16.dp)) {
        Text(item.title, style = MaterialTheme.typography.headlineMedium)
        Spacer(Modifier.height(8.dp))
        Text(item.description)
    }
}
```

---

## メール風UI

```kotlin
@Composable
fun EmailTwoPaneScreen() {
    val isExpanded = currentWindowAdaptiveInfo().windowSizeClass.windowWidthSizeClass == WindowWidthSizeClass.EXPANDED
    var selectedEmail by remember { mutableStateOf<Email?>(null) }

    if (isExpanded) {
        Row(Modifier.fillMaxSize()) {
            EmailList(Modifier.weight(0.35f)) { selectedEmail = it }
            VerticalDivider()
            selectedEmail?.let { email ->
                EmailContent(email, Modifier.weight(0.65f))
            } ?: Box(Modifier.weight(0.65f), contentAlignment = Alignment.Center) {
                Column(horizontalAlignment = Alignment.CenterHorizontally) {
                    Icon(Icons.Default.Email, null, Modifier.size(64.dp), tint = Color.LightGray)
                    Text("メールを選択", color = Color.Gray)
                }
            }
        }
    } else {
        if (selectedEmail != null) {
            EmailContent(selectedEmail!!, Modifier.fillMaxSize())
            BackHandler { selectedEmail = null }
        } else {
            EmailList(Modifier.fillMaxSize()) { selectedEmail = it }
        }
    }
}
```

---

## まとめ

| パターン | 用途 |
|---------|------|
| リスト詳細 | 一覧+詳細の同時表示 |
| `WindowSizeClass` | 画面サイズ判定 |
| `BackHandler` | コンパクト時の戻る |
| `VerticalDivider` | 分割線 |

- `WindowSizeClass`で画面幅に応じたレイアウト切替
- EXPANDED時は横並び、COMPACT時はスタック
- `BackHandler`でコンパクト時の戻る操作
- メール/チャット/設定画面に最適

---

8種類のAndroidアプリテンプレート（レスポンシブ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Adaptive Layout](https://zenn.dev/myougatheaxo/articles/android-compose-compose-adaptive-layout-2026)
- [NavigationRail](https://zenn.dev/myougatheaxo/articles/android-compose-compose-navigation-rail-2026)
- [WindowSizeClass](https://zenn.dev/myougatheaxo/articles/android-compose-compose-window-size-2026)
