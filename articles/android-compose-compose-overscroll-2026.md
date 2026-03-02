---
title: "Overscroll完全ガイド — overScrollMode/カスタムエフェクト/バウンス"
emoji: "🔄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "scroll"]
published: true
---

## この記事で学べること

**Overscroll**（overscrollエフェクト、カスタムバウンス、スクロール端の制御）を解説します。

---

## 標準Overscroll

```kotlin
@Composable
fun StandardOverscrollList() {
    val items = (1..30).toList()

    LazyColumn(
        modifier = Modifier.fillMaxSize(),
        contentPadding = PaddingValues(16.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        items(items) { item ->
            Card(Modifier.fillMaxWidth()) {
                Text("Item $item", Modifier.padding(16.dp))
            }
        }
    }
}
```

---

## カスタムOverscrollエフェクト

```kotlin
@OptIn(ExperimentalFoundationApi::class)
@Composable
fun CustomOverscrollEffect() {
    val scrollState = rememberScrollState()
    var overscrollOffset by remember { mutableFloatStateOf(0f) }

    val overscrollEffect = remember {
        object : OverscrollEffect {
            override fun applyToScroll(
                delta: Offset, source: NestedScrollSource,
                performScroll: (Offset) -> Offset
            ): Offset {
                val consumed = performScroll(delta)
                val remaining = delta - consumed
                overscrollOffset += remaining.y * 0.3f
                return consumed
            }

            override suspend fun applyToFling(
                velocity: Velocity,
                performFling: suspend (Velocity) -> Velocity
            ) {
                performFling(velocity)
                animate(overscrollOffset, 0f) { value, _ ->
                    overscrollOffset = value
                }
            }

            override val isInProgress: Boolean
                get() = overscrollOffset != 0f

            override val effectModifier: Modifier
                get() = Modifier.graphicsLayer {
                    translationY = overscrollOffset
                }
        }
    }

    Column(
        Modifier
            .fillMaxSize()
            .overscroll(overscrollEffect)
            .verticalScroll(scrollState)
            .padding(16.dp)
    ) {
        repeat(20) { index ->
            Card(Modifier.fillMaxWidth().padding(vertical = 4.dp)) {
                Text("Item ${index + 1}", Modifier.padding(16.dp))
            }
        }
    }
}
```

---

## Overscroll無効化

```kotlin
@Composable
fun NoOverscrollList() {
    val items = (1..30).toList()

    CompositionLocalProvider(
        LocalOverscrollConfiguration provides null
    ) {
        LazyColumn(
            modifier = Modifier.fillMaxSize(),
            contentPadding = PaddingValues(16.dp)
        ) {
            items(items) { item ->
                ListItem(headlineContent = { Text("Item $item") })
            }
        }
    }
}
```

---

## まとめ

| 設定 | 用途 |
|------|------|
| デフォルト | プラットフォーム標準エフェクト |
| `OverscrollEffect` | カスタムエフェクト |
| `LocalOverscrollConfiguration` | 無効化 |
| `overscroll()` | エフェクト適用 |

- Android 12+はストレッチ、それ以前はグロー
- `OverscrollEffect`でカスタムバウンス実装
- `LocalOverscrollConfiguration`で無効化可能
- NestedScrollと連携してスクロール制御

---

8種類のAndroidアプリテンプレート（スクロールUI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [NestedScroll](https://zenn.dev/myougatheaxo/articles/android-compose-compose-nested-scroll-2026)
- [PullToRefresh](https://zenn.dev/myougatheaxo/articles/android-compose-compose-pull-refresh-2026)
- [SnapFling](https://zenn.dev/myougatheaxo/articles/android-compose-compose-snap-fling-2026)
