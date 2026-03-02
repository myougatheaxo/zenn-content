---
title: "共有要素遷移完全ガイド — SharedTransitionLayout/AnimatedContent/Navigation"
emoji: "✨"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "animation"]
published: true
---

## この記事で学べること

**共有要素遷移**（SharedTransitionLayout、AnimatedContent、Navigation連携、カスタムアニメーション）を解説します。

---

## SharedTransitionLayout

```kotlin
@Composable
fun SharedTransitionExample() {
    var showDetail by remember { mutableStateOf(false) }
    var selectedItem by remember { mutableStateOf<Item?>(null) }

    SharedTransitionLayout {
        AnimatedContent(targetState = showDetail, label = "transition") { isDetail ->
            if (!isDetail) {
                ItemList(
                    onItemClick = { item ->
                        selectedItem = item
                        showDetail = true
                    },
                    animatedVisibilityScope = this@AnimatedContent,
                    sharedTransitionScope = this@SharedTransitionLayout
                )
            } else {
                selectedItem?.let { item ->
                    ItemDetail(
                        item = item,
                        onBack = { showDetail = false },
                        animatedVisibilityScope = this@AnimatedContent,
                        sharedTransitionScope = this@SharedTransitionLayout
                    )
                }
            }
        }
    }
}
```

---

## リスト→詳細遷移

```kotlin
@Composable
fun SharedTransitionScope.ItemList(
    onItemClick: (Item) -> Unit,
    animatedVisibilityScope: AnimatedVisibilityScope,
    sharedTransitionScope: SharedTransitionScope
) {
    LazyColumn {
        items(sampleItems) { item ->
            Row(
                Modifier
                    .fillMaxWidth()
                    .clickable { onItemClick(item) }
                    .padding(16.dp)
            ) {
                AsyncImage(
                    model = item.imageUrl,
                    contentDescription = null,
                    modifier = Modifier
                        .size(60.dp)
                        .sharedElement(
                            rememberSharedContentState(key = "image_${item.id}"),
                            animatedVisibilityScope = animatedVisibilityScope
                        )
                )
                Spacer(Modifier.width(16.dp))
                Text(
                    item.title,
                    modifier = Modifier.sharedElement(
                        rememberSharedContentState(key = "title_${item.id}"),
                        animatedVisibilityScope = animatedVisibilityScope
                    )
                )
            }
        }
    }
}

@Composable
fun SharedTransitionScope.ItemDetail(
    item: Item,
    onBack: () -> Unit,
    animatedVisibilityScope: AnimatedVisibilityScope,
    sharedTransitionScope: SharedTransitionScope
) {
    Column(Modifier.fillMaxSize()) {
        AsyncImage(
            model = item.imageUrl,
            contentDescription = null,
            modifier = Modifier
                .fillMaxWidth()
                .height(300.dp)
                .sharedElement(
                    rememberSharedContentState(key = "image_${item.id}"),
                    animatedVisibilityScope = animatedVisibilityScope
                )
        )
        Text(
            item.title,
            style = MaterialTheme.typography.headlineMedium,
            modifier = Modifier
                .padding(16.dp)
                .sharedElement(
                    rememberSharedContentState(key = "title_${item.id}"),
                    animatedVisibilityScope = animatedVisibilityScope
                )
        )
        Text(item.description, Modifier.padding(horizontal = 16.dp))
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| SharedTransitionLayout | 遷移スコープ |
| sharedElement | 共有要素指定 |
| AnimatedContent | 画面切替 |
| rememberSharedContentState | 要素キー |

- `SharedTransitionLayout`で共有要素遷移のスコープを作成
- `sharedElement`で同じキーの要素をアニメーション接続
- `AnimatedContent`で画面遷移をトリガー
- 画像とテキストが滑らかに移動するUX

---

8種類のAndroidアプリテンプレート（アニメーション対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [アニメーション](https://zenn.dev/myougatheaxo/articles/android-compose-animation-spring-2026)
- [Navigation](https://zenn.dev/myougatheaxo/articles/android-compose-type-safe-navigation-2026)
- [Coil画像](https://zenn.dev/myougatheaxo/articles/android-compose-coil-image-2026)
