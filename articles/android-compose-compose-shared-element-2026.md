---
title: "Compose SharedElement完全ガイド — 共有要素遷移/SharedTransitionLayout/アニメーション"
emoji: "🔀"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "animation"]
published: true
---

## この記事で学べること

**Compose SharedElement**（SharedTransitionLayout、共有要素遷移、リスト→詳細アニメーション）を解説します。

---

## 基本SharedElement

```kotlin
@Composable
fun SharedElementDemo() {
    var selectedItem by remember { mutableStateOf<Item?>(null) }

    SharedTransitionLayout {
        AnimatedContent(targetState = selectedItem, label = "shared") { item ->
            if (item == null) {
                ItemList(
                    onItemClick = { selectedItem = it },
                    animatedVisibilityScope = this@AnimatedContent,
                    sharedTransitionScope = this@SharedTransitionLayout
                )
            } else {
                ItemDetail(
                    item = item,
                    onBack = { selectedItem = null },
                    animatedVisibilityScope = this@AnimatedContent,
                    sharedTransitionScope = this@SharedTransitionLayout
                )
            }
        }
    }
}
```

---

## リスト画面

```kotlin
@Composable
fun SharedTransitionScope.ItemList(
    onItemClick: (Item) -> Unit,
    animatedVisibilityScope: AnimatedVisibilityScope,
    sharedTransitionScope: SharedTransitionScope
) {
    LazyColumn(Modifier.fillMaxSize().padding(16.dp), verticalArrangement = Arrangement.spacedBy(8.dp)) {
        items(sampleItems) { item ->
            Card(Modifier.fillMaxWidth().clickable { onItemClick(item) }) {
                Row(Modifier.padding(12.dp), verticalAlignment = Alignment.CenterVertically) {
                    AsyncImage(
                        model = item.imageUrl,
                        contentDescription = null,
                        modifier = Modifier
                            .size(60.dp)
                            .clip(RoundedCornerShape(8.dp))
                            .sharedElement(
                                rememberSharedContentState(key = "image-${item.id}"),
                                animatedVisibilityScope = animatedVisibilityScope
                            )
                    )
                    Spacer(Modifier.width(12.dp))
                    Text(
                        item.title,
                        modifier = Modifier.sharedElement(
                            rememberSharedContentState(key = "title-${item.id}"),
                            animatedVisibilityScope = animatedVisibilityScope
                        )
                    )
                }
            }
        }
    }
}
```

---

## 詳細画面

```kotlin
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
                    rememberSharedContentState(key = "image-${item.id}"),
                    animatedVisibilityScope = animatedVisibilityScope
                ),
            contentScale = ContentScale.Crop
        )
        Text(
            item.title,
            style = MaterialTheme.typography.headlineMedium,
            modifier = Modifier
                .padding(16.dp)
                .sharedElement(
                    rememberSharedContentState(key = "title-${item.id}"),
                    animatedVisibilityScope = animatedVisibilityScope
                )
        )
        Text(item.description, Modifier.padding(horizontal = 16.dp))

        TextButton(onClick = onBack, Modifier.padding(16.dp)) { Text("戻る") }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `SharedTransitionLayout` | 共有遷移スコープ |
| `sharedElement` | 共有要素指定 |
| `rememberSharedContentState` | 要素キー管理 |
| `AnimatedContent` | 遷移アニメーション |

- `SharedTransitionLayout`でスコープを作成
- 同じ`key`の要素が画面間でアニメーション遷移
- リスト→詳細の画像/テキスト共有遷移に最適
- `AnimatedContent`と組み合わせて使用

---

8種類のAndroidアプリテンプレート（アニメーション対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Animation](https://zenn.dev/myougatheaxo/articles/android-compose-compose-animation-2026)
- [Compose PredictiveBack](https://zenn.dev/myougatheaxo/articles/android-compose-compose-predictive-back-2026)
- [Navigation TypeSafe](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-type-safe-2026)
