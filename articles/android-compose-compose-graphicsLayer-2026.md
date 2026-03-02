---
title: "Compose graphicsLayer完全ガイド — 回転/スケール/透明度/3D変換"
emoji: "🔄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**Compose graphicsLayer**（回転、スケール、透明度、3D変換、カメラ距離）を解説します。

---

## 基本graphicsLayer

```kotlin
@Composable
fun GraphicsLayerDemo() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(16.dp)) {
        // 回転
        Text(
            "45度回転",
            Modifier.graphicsLayer { rotationZ = 45f }
        )

        // スケール
        Text(
            "1.5倍",
            Modifier.graphicsLayer { scaleX = 1.5f; scaleY = 1.5f }
        )

        // 透明度
        Text(
            "半透明",
            Modifier.graphicsLayer { alpha = 0.5f }
        )

        // 移動
        Text(
            "移動",
            Modifier.graphicsLayer { translationX = 50f; translationY = 20f }
        )
    }
}
```

---

## 3D変換

```kotlin
@Composable
fun Card3DFlip() {
    var flipped by remember { mutableStateOf(false) }
    val rotation by animateFloatAsState(
        targetValue = if (flipped) 180f else 0f,
        animationSpec = tween(600), label = "flip"
    )

    Box(
        Modifier
            .size(200.dp, 120.dp)
            .graphicsLayer {
                rotationY = rotation
                cameraDistance = 12f * density
            }
            .clickable { flipped = !flipped },
        contentAlignment = Alignment.Center
    ) {
        if (rotation <= 90f) {
            Card(Modifier.fillMaxSize(), colors = CardDefaults.cardColors(containerColor = Color(0xFF2196F3))) {
                Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                    Text("表面", color = Color.White, style = MaterialTheme.typography.headlineSmall)
                }
            }
        } else {
            Card(
                Modifier.fillMaxSize().graphicsLayer { rotationY = 180f },
                colors = CardDefaults.cardColors(containerColor = Color(0xFFFF9800))
            ) {
                Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                    Text("裏面", color = Color.White, style = MaterialTheme.typography.headlineSmall)
                }
            }
        }
    }
}
```

---

## パフォーマンス最適化

```kotlin
@Composable
fun OptimizedScroll(scrollState: LazyListState) {
    val firstVisibleOffset by remember {
        derivedStateOf { scrollState.firstVisibleItemScrollOffset }
    }

    Image(
        painter = painterResource(R.drawable.header),
        contentDescription = null,
        modifier = Modifier
            .fillMaxWidth()
            .height(200.dp)
            .graphicsLayer {
                // recompositionなしで変換（高パフォーマンス）
                alpha = 1f - (firstVisibleOffset / 500f).coerceIn(0f, 1f)
                translationY = firstVisibleOffset * 0.5f
            },
        contentScale = ContentScale.Crop
    )
}
```

---

## まとめ

| プロパティ | 用途 |
|-----------|------|
| `rotationZ/X/Y` | 回転（2D/3D） |
| `scaleX/Y` | スケール |
| `alpha` | 透明度 |
| `cameraDistance` | 3D遠近感 |

- `graphicsLayer`はrecompositionなしで変換可能
- `cameraDistance`で3Dフリップの遠近感調整
- スクロール連動アニメーションに最適
- `derivedStateOf`と組み合わせてパフォーマンス最適化

---

8種類のAndroidアプリテンプレート（カスタムUI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Shadow](https://zenn.dev/myougatheaxo/articles/android-compose-compose-shadow-2026)
- [Compose Animation](https://zenn.dev/myougatheaxo/articles/android-compose-compose-animation-2026)
- [Compose Canvas](https://zenn.dev/myougatheaxo/articles/android-compose-compose-canvas-2026)
