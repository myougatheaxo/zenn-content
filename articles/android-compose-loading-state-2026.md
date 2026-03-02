---
title: "ローディング状態管理ガイド — Shimmer/スケルトン/プレースホルダー"
emoji: "⏳"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "loading"]
published: true
---

## この記事で学べること

Composeでの**ローディング表示パターン**（Shimmer、スケルトン、プレースホルダー）を解説します。

---

## 基本のローディング状態

```kotlin
sealed interface UiState<out T> {
    data object Loading : UiState<Nothing>
    data class Success<T>(val data: T) : UiState<T>
    data class Error(val message: String) : UiState<Nothing>
}

@Composable
fun <T> LoadingContent(
    state: UiState<T>,
    onRetry: () -> Unit,
    content: @Composable (T) -> Unit
) {
    when (state) {
        is UiState.Loading -> LoadingView()
        is UiState.Success -> content(state.data)
        is UiState.Error -> ErrorView(state.message, onRetry)
    }
}
```

---

## Shimmerエフェクト

```kotlin
@Composable
fun ShimmerEffect(modifier: Modifier = Modifier) {
    val transition = rememberInfiniteTransition(label = "shimmer")
    val translateAnim by transition.animateFloat(
        initialValue = 0f,
        targetValue = 1000f,
        animationSpec = infiniteRepeatable(
            animation = tween(1200, easing = LinearEasing),
            repeatMode = RepeatMode.Restart
        ),
        label = "shimmer"
    )

    val brush = Brush.linearGradient(
        colors = listOf(
            Color.LightGray.copy(alpha = 0.6f),
            Color.LightGray.copy(alpha = 0.2f),
            Color.LightGray.copy(alpha = 0.6f)
        ),
        start = Offset(translateAnim - 200f, translateAnim - 200f),
        end = Offset(translateAnim, translateAnim)
    )

    Box(
        modifier = modifier
            .clip(RoundedCornerShape(8.dp))
            .background(brush)
    )
}
```

---

## スケルトンスクリーン

```kotlin
@Composable
fun SkeletonListItem() {
    Row(
        Modifier
            .fillMaxWidth()
            .padding(16.dp),
        verticalAlignment = Alignment.CenterVertically
    ) {
        // アバター
        ShimmerEffect(
            Modifier
                .size(48.dp)
                .clip(CircleShape)
        )
        Spacer(Modifier.width(16.dp))
        Column(Modifier.weight(1f)) {
            // タイトル
            ShimmerEffect(
                Modifier
                    .fillMaxWidth(0.7f)
                    .height(16.dp)
            )
            Spacer(Modifier.height(8.dp))
            // サブタイトル
            ShimmerEffect(
                Modifier
                    .fillMaxWidth(0.5f)
                    .height(12.dp)
            )
        }
    }
}

@Composable
fun SkeletonList(count: Int = 5) {
    LazyColumn {
        items(count) {
            SkeletonListItem()
        }
    }
}
```

---

## カードスケルトン

```kotlin
@Composable
fun SkeletonCard() {
    Card(
        Modifier
            .fillMaxWidth()
            .padding(8.dp)
    ) {
        Column {
            // 画像プレースホルダー
            ShimmerEffect(
                Modifier
                    .fillMaxWidth()
                    .height(180.dp)
            )
            Column(Modifier.padding(16.dp)) {
                ShimmerEffect(Modifier.fillMaxWidth(0.8f).height(20.dp))
                Spacer(Modifier.height(8.dp))
                ShimmerEffect(Modifier.fillMaxWidth(0.6f).height(14.dp))
                Spacer(Modifier.height(4.dp))
                ShimmerEffect(Modifier.fillMaxWidth(0.9f).height(14.dp))
            }
        }
    }
}
```

---

## Crossfadeでの切り替え

```kotlin
@Composable
fun <T> AnimatedLoadingContent(
    state: UiState<T>,
    onRetry: () -> Unit,
    content: @Composable (T) -> Unit
) {
    Crossfade(targetState = state, label = "loading") { currentState ->
        when (currentState) {
            is UiState.Loading -> SkeletonList()
            is UiState.Success -> content(currentState.data)
            is UiState.Error -> ErrorView(currentState.message, onRetry)
        }
    }
}
```

---

## ボタンのローディング

```kotlin
@Composable
fun LoadingButton(
    text: String,
    isLoading: Boolean,
    onClick: () -> Unit,
    modifier: Modifier = Modifier
) {
    Button(
        onClick = onClick,
        enabled = !isLoading,
        modifier = modifier
    ) {
        if (isLoading) {
            CircularProgressIndicator(
                modifier = Modifier.size(20.dp),
                color = MaterialTheme.colorScheme.onPrimary,
                strokeWidth = 2.dp
            )
            Spacer(Modifier.width(8.dp))
        }
        Text(if (isLoading) "処理中..." else text)
    }
}
```

---

## まとめ

- `UiState` sealed interfaceで状態管理
- Shimmerエフェクトで`infiniteRepeatable`アニメーション
- スケルトンスクリーンで実際のUIに近いプレースホルダー
- `Crossfade`でスムーズな状態切り替え
- ボタンのローディング状態で二重送信防止
- リスト・カードそれぞれのスケルトンパターン

---

8種類のAndroidアプリテンプレート（ローディング表示設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [エラーハンドリングUIガイド](https://zenn.dev/myougatheaxo/articles/android-compose-error-handling-ui-2026)
- [アニメーション完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-animation-advanced-2026)
- [Shimmerローディング](https://zenn.dev/myougatheaxo/articles/android-compose-shimmer-loading-2026)
