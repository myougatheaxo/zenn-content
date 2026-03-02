---
title: "LazyColumn パフォーマンス最適化 — key/contentType/安定マーカーで高速化"
emoji: "⚡"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "performance"]
published: true
---

## この記事で学べること

LazyColumnの**パフォーマンス最適化テクニック**（key、contentType、安定マーカー、リコンポジション制御）を解説します。

---

## key の指定

```kotlin
// NG: keyなし → アイテム移動/削除時に全リコンポーズ
LazyColumn {
    items(items) { item -> ItemRow(item) }
}

// OK: key指定 → アイテム単位で差分更新
LazyColumn {
    items(items, key = { it.id }) { item ->
        ItemRow(item)
    }
}
```

---

## contentType

```kotlin
// 異なるタイプのアイテムでRecyclerPool効率化
LazyColumn {
    items(
        items = mixedItems,
        key = { it.id },
        contentType = { it.type } // "header", "item", "ad"
    ) { item ->
        when (item.type) {
            "header" -> HeaderRow(item)
            "item" -> ItemRow(item)
            "ad" -> AdBanner(item)
        }
    }
}
```

---

## @Stable と @Immutable

```kotlin
// @Immutable: プロパティが絶対に変わらないデータ
@Immutable
data class ProductItem(
    val id: String,
    val name: String,
    val price: Int,
    val imageUrl: String
)

// @Stable: 変更は追跡可能（MutableStateで）
@Stable
class CartState {
    var itemCount by mutableIntStateOf(0)
    var total by mutableIntStateOf(0)
}

// Composeはこれらのマーカーでスマートリコンポーズを最適化
@Composable
fun ProductRow(product: ProductItem) { // @Immutableなので同値なら再コンポーズスキップ
    ListItem(
        headlineContent = { Text(product.name) },
        trailingContent = { Text("¥${product.price}") }
    )
}
```

---

## ラムダのキャッシュ

```kotlin
// NG: 毎リコンポジションで新しいラムダ生成
@Composable
fun ItemList(items: List<Item>, viewModel: ItemViewModel) {
    LazyColumn {
        items(items, key = { it.id }) { item ->
            ItemRow(
                item = item,
                onClick = { viewModel.selectItem(item.id) } // 毎回新しいラムダ
            )
        }
    }
}

// OK: rememberでラムダをキャッシュ
@Composable
fun ItemList(items: List<Item>, viewModel: ItemViewModel) {
    LazyColumn {
        items(items, key = { it.id }) { item ->
            val onClick = remember(item.id) {
                { viewModel.selectItem(item.id) }
            }
            ItemRow(item = item, onClick = onClick)
        }
    }
}
```

---

## prefetchの最適化

```kotlin
@Composable
fun OptimizedList(items: List<Item>) {
    val listState = rememberLazyListState()

    LazyColumn(
        state = listState,
        // prefetchの最適化は自動だが、余白設定で改善
        contentPadding = PaddingValues(vertical = 8.dp),
        verticalArrangement = Arrangement.spacedBy(4.dp)
    ) {
        items(
            items = items,
            key = { it.id },
            contentType = { "item" }
        ) { item ->
            // 重いComposableを避ける
            // SubcomposeAsyncImageよりAsyncImageが軽量
            ItemRow(item)
        }
    }
}
```

---

## derivedStateOf でリコンポーズ削減

```kotlin
@Composable
fun ScrollAwareList(items: List<Item>) {
    val listState = rememberLazyListState()

    // NG: firstVisibleItemIndexが変わるたびにリコンポーズ
    // val showButton = listState.firstVisibleItemIndex > 5

    // OK: derivedStateOfで必要な時だけリコンポーズ
    val showButton by remember {
        derivedStateOf { listState.firstVisibleItemIndex > 5 }
    }

    Box {
        LazyColumn(state = listState) {
            items(items, key = { it.id }) { ItemRow(it) }
        }

        if (showButton) {
            FloatingActionButton(
                onClick = { /* scroll to top */ },
                modifier = Modifier.align(Alignment.BottomEnd).padding(16.dp)
            ) {
                Icon(Icons.Default.ArrowUpward, "トップへ")
            }
        }
    }
}
```

---

## チェックリスト

| 項目 | 効果 |
|------|------|
| `key = { it.id }` | 差分更新でアイテム移動/削除が高速 |
| `contentType` | ViewHolder再利用の効率化 |
| `@Immutable`/`@Stable` | 不要なリコンポーズをスキップ |
| ラムダのrememberキャッシュ | ラムダ生成による再コンポーズ防止 |
| `derivedStateOf` | 派生状態の変更時のみリコンポーズ |
| `AsyncImage`（`Sub-`ではない） | 軽量な画像読み込み |

---

## まとめ

- `key`は**必須** — すべてのLazyColumn/LazyRowに指定
- `contentType`で異なるアイテムタイプの再利用効率化
- `@Immutable`/`@Stable`でスマートリコンポーズ
- ラムダは`remember`でキャッシュ
- `derivedStateOf`で不要なリコンポーズ削減
- レイアウトインスペクターで実際のリコンポーズ回数を確認

---

8種類のAndroidアプリテンプレート（パフォーマンス最適化済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [LazyColumn完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-lazycolumn-guide-2026)
- [remember完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-remember-patterns-2026)
- [無限スクロール実装](https://zenn.dev/myougatheaxo/articles/android-compose-infinite-scroll-2026)
