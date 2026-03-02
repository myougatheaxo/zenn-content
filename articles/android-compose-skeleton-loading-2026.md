---
title: "スケルトンローディング完全ガイド — ContentType別/リスト/グリッド"
emoji: "💀"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**スケルトンローディング**（ContentType別スケルトン、リスト/グリッド対応、Modifier拡張、UiState統合）を解説します。

---

## 汎用スケルトンModifier

```kotlin
fun Modifier.skeleton(
    isLoading: Boolean,
    shape: Shape = RoundedCornerShape(4.dp)
): Modifier = composed {
    if (!isLoading) return@composed this

    val transition = rememberInfiniteTransition(label = "skeleton")
    val alpha by transition.animateFloat(
        initialValue = 0.3f,
        targetValue = 0.7f,
        animationSpec = infiniteRepeatable(
            animation = tween(800),
            repeatMode = RepeatMode.Reverse
        ),
        label = "skeletonAlpha"
    )

    this
        .clip(shape)
        .drawWithContent {
            drawRoundRect(
                color = Color.Gray.copy(alpha = alpha),
                cornerRadius = CornerRadius(4.dp.toPx())
            )
        }
}
```

---

## カードスケルトン

```kotlin
@Composable
fun ProductCardSkeleton() {
    Card(Modifier.fillMaxWidth().padding(8.dp)) {
        Column(Modifier.padding(12.dp)) {
            // 画像プレースホルダー
            Box(
                Modifier.fillMaxWidth().height(150.dp)
                    .skeleton(true, RoundedCornerShape(8.dp))
            )
            Spacer(Modifier.height(12.dp))
            // タイトル
            Box(Modifier.fillMaxWidth(0.7f).height(18.dp).skeleton(true))
            Spacer(Modifier.height(8.dp))
            // 説明文
            Box(Modifier.fillMaxWidth().height(14.dp).skeleton(true))
            Spacer(Modifier.height(4.dp))
            Box(Modifier.fillMaxWidth(0.5f).height(14.dp).skeleton(true))
            Spacer(Modifier.height(12.dp))
            // 価格
            Row(Modifier.fillMaxWidth(), horizontalArrangement = Arrangement.SpaceBetween) {
                Box(Modifier.width(80.dp).height(20.dp).skeleton(true))
                Box(Modifier.width(40.dp).height(20.dp).skeleton(true, CircleShape))
            }
        }
    }
}
```

---

## グリッドスケルトン

```kotlin
@Composable
fun ProductGridSkeleton(columns: Int = 2, itemCount: Int = 6) {
    LazyVerticalGrid(
        columns = GridCells.Fixed(columns),
        contentPadding = PaddingValues(8.dp),
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        items(itemCount) {
            ProductCardSkeleton()
        }
    }
}
```

---

## UiState統合

```kotlin
@Composable
fun <T> SkeletonContent(
    state: UiState<T>,
    skeleton: @Composable () -> Unit,
    error: @Composable (String) -> Unit,
    content: @Composable (T) -> Unit
) {
    Crossfade(targetState = state, label = "skeleton") { currentState ->
        when (currentState) {
            is UiState.Loading -> skeleton()
            is UiState.Error -> error(currentState.message)
            is UiState.Success -> content(currentState.data)
        }
    }
}

// 使用
@Composable
fun ProductListScreen(viewModel: ProductViewModel = hiltViewModel()) {
    val uiState by viewModel.products.collectAsStateWithLifecycle()

    SkeletonContent(
        state = uiState,
        skeleton = { ProductGridSkeleton() },
        error = { ErrorContent(it) { viewModel.retry() } },
        content = { products -> ProductGrid(products) }
    )
}
```

---

## まとめ

| パターン | 用途 |
|----------|------|
| `skeleton()` Modifier | 汎用ぼかし効果 |
| カードスケルトン | コンテンツ形状の模倣 |
| グリッドスケルトン | 一覧画面 |
| `SkeletonContent` | UiState統合 |

- `Modifier.skeleton()`で任意のComposableをスケルトン化
- コンテンツと同じレイアウトでスケルトン作成
- `Crossfade`でスムーズな切替
- `UiState`に応じて自動的にスケルトン/コンテンツ表示

---

8種類のAndroidアプリテンプレート（ローディングUI実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [シマー効果](https://zenn.dev/myougatheaxo/articles/android-compose-shimmer-placeholder-2026)
- [エラーUI](https://zenn.dev/myougatheaxo/articles/android-compose-error-handling-ui-2026)
- [LazyColumn](https://zenn.dev/myougatheaxo/articles/android-compose-lazy-column-optimization-2026)
