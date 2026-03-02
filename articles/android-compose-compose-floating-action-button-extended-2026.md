---
title: "Compose ExtendedFAB完全ガイド — アニメーション/スクロール連動/カスタムFAB"
emoji: "➕"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**Compose ExtendedFAB**（ExtendedFloatingActionButton、スクロール連動、アニメーション、FABバリエーション）を解説します。

---

## FABバリエーション

```kotlin
@Composable
fun FabVariantsDemo() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(16.dp)) {
        // Small FAB
        SmallFloatingActionButton(onClick = {}) {
            Icon(Icons.Default.Add, "追加")
        }

        // 標準 FAB
        FloatingActionButton(onClick = {}) {
            Icon(Icons.Default.Edit, "編集")
        }

        // Large FAB
        LargeFloatingActionButton(onClick = {}) {
            Icon(Icons.Default.Add, "追加", Modifier.size(36.dp))
        }

        // Extended FAB（テキスト付き）
        ExtendedFloatingActionButton(
            onClick = {},
            icon = { Icon(Icons.Default.Create, "作成") },
            text = { Text("新規作成") }
        )
    }
}
```

---

## スクロール連動Extended FAB

```kotlin
@Composable
fun ScrollAwareExtendedFab() {
    val listState = rememberLazyListState()
    val expanded by remember { derivedStateOf { listState.firstVisibleItemIndex == 0 } }

    Scaffold(
        floatingActionButton = {
            ExtendedFloatingActionButton(
                onClick = { /* 新規作成 */ },
                expanded = expanded,
                icon = { Icon(Icons.Default.Add, "追加") },
                text = { Text("新規作成") }
            )
        }
    ) { padding ->
        LazyColumn(state = listState, modifier = Modifier.padding(padding)) {
            items(50) { index ->
                ListItem(
                    headlineContent = { Text("アイテム $index") },
                    leadingContent = { Icon(Icons.Default.Article, null) }
                )
                HorizontalDivider()
            }
        }
    }
}
```

---

## カスタムカラーFAB

```kotlin
@Composable
fun CustomFabDemo() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(16.dp)) {
        // Primary（デフォルト）
        FloatingActionButton(onClick = {}) {
            Icon(Icons.Default.Add, "追加")
        }

        // Secondary
        FloatingActionButton(
            onClick = {},
            containerColor = MaterialTheme.colorScheme.secondaryContainer,
            contentColor = MaterialTheme.colorScheme.onSecondaryContainer
        ) { Icon(Icons.Default.Favorite, "お気に入り") }

        // Tertiary
        FloatingActionButton(
            onClick = {},
            containerColor = MaterialTheme.colorScheme.tertiaryContainer,
            contentColor = MaterialTheme.colorScheme.onTertiaryContainer
        ) { Icon(Icons.Default.Chat, "チャット") }

        // Surface（M3 Surface variant）
        FloatingActionButton(
            onClick = {},
            containerColor = MaterialTheme.colorScheme.surface,
            contentColor = MaterialTheme.colorScheme.primary,
            elevation = FloatingActionButtonDefaults.elevation(defaultElevation = 4.dp)
        ) { Icon(Icons.Default.Search, "検索") }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `FloatingActionButton` | 標準FAB |
| `SmallFloatingActionButton` | 小サイズFAB |
| `LargeFloatingActionButton` | 大サイズFAB |
| `ExtendedFloatingActionButton` | テキスト付きFAB |

- M3では4種類のFABサイズ（Small/Standard/Large/Extended）
- `expanded`パラメータでExtendedの展開/収縮を制御
- `derivedStateOf`でスクロール位置からFAB状態を導出
- `containerColor`/`contentColor`でカラーバリエーション

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose IconButton](https://zenn.dev/myougatheaxo/articles/android-compose-compose-icon-button-2026)
- [Compose BottomAppBar](https://zenn.dev/myougatheaxo/articles/android-compose-compose-bottom-app-bar-2026)
- [Compose ScrollState](https://zenn.dev/myougatheaxo/articles/android-compose-compose-scroll-state-2026)
