---
title: "Composeアニメーション実践 — AnimatedVisibility/Crossfade/updateTransition"
emoji: "✨"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "animation"]
published: true
---

## この記事で学べること

Composeの**アニメーション**実践パターン（AnimatedVisibility、Crossfade、updateTransition、animateContentSize）を解説します。

---

## AnimatedVisibility

```kotlin
@Composable
fun AnimatedPanel() {
    var visible by remember { mutableStateOf(false) }

    Column {
        Button(onClick = { visible = !visible }) {
            Text(if (visible) "非表示" else "表示")
        }

        AnimatedVisibility(
            visible = visible,
            enter = slideInVertically() + fadeIn(),
            exit = slideOutVertically() + fadeOut()
        ) {
            Card(
                Modifier
                    .fillMaxWidth()
                    .padding(16.dp)
            ) {
                Text("アニメーション付きパネル", Modifier.padding(24.dp))
            }
        }
    }
}

// カスタムアニメーション
AnimatedVisibility(
    visible = visible,
    enter = slideInHorizontally(
        initialOffsetX = { fullWidth -> -fullWidth },
        animationSpec = tween(500, easing = EaseOutCubic)
    ) + expandHorizontally() + fadeIn(),
    exit = slideOutHorizontally(
        targetOffsetX = { fullWidth -> fullWidth }
    ) + shrinkHorizontally() + fadeOut()
) {
    // コンテンツ
}
```

---

## Crossfade（コンテンツ切替）

```kotlin
@Composable
fun TabContent(selectedTab: Int) {
    Crossfade(
        targetState = selectedTab,
        animationSpec = tween(300),
        label = "tab_crossfade"
    ) { tab ->
        when (tab) {
            0 -> HomeContent()
            1 -> SearchContent()
            2 -> ProfileContent()
        }
    }
}

// AnimatedContent（より柔軟）
@Composable
fun CounterDisplay(count: Int) {
    AnimatedContent(
        targetState = count,
        transitionSpec = {
            if (targetState > initialState) {
                slideInVertically { -it } + fadeIn() togetherWith
                    slideOutVertically { it } + fadeOut()
            } else {
                slideInVertically { it } + fadeIn() togetherWith
                    slideOutVertically { -it } + fadeOut()
            }.using(SizeTransform(clip = false))
        },
        label = "counter"
    ) { targetCount ->
        Text(
            text = "$targetCount",
            style = MaterialTheme.typography.displayLarge
        )
    }
}
```

---

## updateTransition（複合アニメーション）

```kotlin
enum class CardState { Normal, Expanded, Selected }

@Composable
fun AnimatedCard(state: CardState) {
    val transition = updateTransition(targetState = state, label = "card")

    val elevation by transition.animateDp(label = "elevation") { s ->
        when (s) {
            CardState.Normal -> 2.dp
            CardState.Expanded -> 8.dp
            CardState.Selected -> 4.dp
        }
    }

    val color by transition.animateColor(label = "color") { s ->
        when (s) {
            CardState.Normal -> MaterialTheme.colorScheme.surface
            CardState.Expanded -> MaterialTheme.colorScheme.surfaceVariant
            CardState.Selected -> MaterialTheme.colorScheme.primaryContainer
        }
    }

    val cornerRadius by transition.animateDp(label = "corner") { s ->
        when (s) {
            CardState.Normal -> 8.dp
            CardState.Expanded -> 16.dp
            CardState.Selected -> 12.dp
        }
    }

    Card(
        modifier = Modifier.fillMaxWidth().padding(8.dp),
        elevation = CardDefaults.cardElevation(elevation),
        colors = CardDefaults.cardColors(containerColor = color),
        shape = RoundedCornerShape(cornerRadius)
    ) {
        // コンテンツ
    }
}
```

---

## animateContentSize

```kotlin
@Composable
fun ExpandableText(text: String, maxLines: Int = 3) {
    var expanded by remember { mutableStateOf(false) }

    Column(
        modifier = Modifier
            .animateContentSize(animationSpec = spring(dampingRatio = Spring.DampingRatioLowBouncy))
            .clickable { expanded = !expanded }
    ) {
        Text(
            text = text,
            maxLines = if (expanded) Int.MAX_VALUE else maxLines,
            overflow = TextOverflow.Ellipsis
        )

        Text(
            text = if (expanded) "閉じる ▲" else "もっと見る ▼",
            color = MaterialTheme.colorScheme.primary,
            style = MaterialTheme.typography.labelMedium
        )
    }
}
```

---

## 無限アニメーション

```kotlin
@Composable
fun PulsingDot() {
    val infiniteTransition = rememberInfiniteTransition(label = "pulse")

    val scale by infiniteTransition.animateFloat(
        initialValue = 1f,
        targetValue = 1.3f,
        animationSpec = infiniteRepeatable(
            animation = tween(600, easing = EaseInOut),
            repeatMode = RepeatMode.Reverse
        ),
        label = "scale"
    )

    val alpha by infiniteTransition.animateFloat(
        initialValue = 1f,
        targetValue = 0.5f,
        animationSpec = infiniteRepeatable(
            animation = tween(600),
            repeatMode = RepeatMode.Reverse
        ),
        label = "alpha"
    )

    Box(
        Modifier
            .size(20.dp)
            .graphicsLayer(scaleX = scale, scaleY = scale, alpha = alpha)
            .background(Color.Red, CircleShape)
    )
}
```

---

## リストアニメーション

```kotlin
@Composable
fun AnimatedList(items: List<Item>) {
    LazyColumn {
        items(items, key = { it.id }) { item ->
            Row(
                modifier = Modifier
                    .animateItem() // リスト内アイテムのアニメーション
                    .fillMaxWidth()
                    .padding(16.dp)
            ) {
                Text(item.name)
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `AnimatedVisibility` | 表示/非表示切替 |
| `Crossfade` | コンテンツ切替（フェード） |
| `AnimatedContent` | コンテンツ切替（カスタム） |
| `updateTransition` | 複数プロパティ同期アニメーション |
| `animateContentSize` | サイズ変更アニメーション |
| `rememberInfiniteTransition` | 無限ループアニメーション |
| `animateItem` | LazyColumnアイテムアニメーション |

- `enter`/`exit`でカスタム表示・非表示効果
- `spring()`で物理ベースのバウンスアニメーション
- `tween()`で時間ベースのイージングアニメーション
- `graphicsLayer`で描画レイヤー分離（パフォーマンス向上）

---

8種類のAndroidアプリテンプレート（アニメーション実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [AnimationSpec](https://zenn.dev/myougatheaxo/articles/android-compose-animation-spec-2026)
- [Shared Element](https://zenn.dev/myougatheaxo/articles/android-compose-shared-element-2026)
- [Canvas描画](https://zenn.dev/myougatheaxo/articles/android-compose-canvas-drawing-2026)
