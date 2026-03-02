---
title: "Compose Transformable完全ガイド — ピンチズーム/回転/パン/TransformableState"
emoji: "🔄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "gesture"]
published: true
---

## この記事で学べること

**Compose Transformable**（ピンチズーム、回転、パン、TransformableState、制限付きトランスフォーム）を解説します。

---

## 基本のTransformable

```kotlin
@Composable
fun TransformableDemo() {
    var scale by remember { mutableFloatStateOf(1f) }
    var rotation by remember { mutableFloatStateOf(0f) }
    var offset by remember { mutableStateOf(Offset.Zero) }

    val state = rememberTransformableState { zoomChange, panChange, rotationChange ->
        scale = (scale * zoomChange).coerceIn(0.5f, 5f)
        rotation += rotationChange
        offset += panChange
    }

    Box(
        Modifier.fillMaxSize()
            .transformable(state)
            .graphicsLayer {
                scaleX = scale
                scaleY = scale
                rotationZ = rotation
                translationX = offset.x
                translationY = offset.y
            },
        contentAlignment = Alignment.Center
    ) {
        Card(Modifier.size(200.dp)) {
            Box(contentAlignment = Alignment.Center, modifier = Modifier.fillMaxSize()) {
                Text("ピンチ/回転/パン", textAlign = TextAlign.Center)
            }
        }
    }
}
```

---

## 画像ビューアー

```kotlin
@Composable
fun ZoomableImageViewer(imageUrl: String) {
    var scale by remember { mutableFloatStateOf(1f) }
    var offset by remember { mutableStateOf(Offset.Zero) }

    val state = rememberTransformableState { zoomChange, panChange, _ ->
        scale = (scale * zoomChange).coerceIn(1f, 5f)
        // ズーム時のみパン許可
        if (scale > 1f) {
            offset += panChange
        } else {
            offset = Offset.Zero
        }
    }

    Box(Modifier.fillMaxSize().clipToBounds()) {
        AsyncImage(
            model = imageUrl,
            contentDescription = null,
            modifier = Modifier.fillMaxSize()
                .transformable(state)
                .graphicsLayer {
                    scaleX = scale
                    scaleY = scale
                    translationX = offset.x
                    translationY = offset.y
                },
            contentScale = ContentScale.Fit
        )

        // ダブルタップでリセット
        Box(
            Modifier.fillMaxSize()
                .pointerInput(Unit) {
                    detectTapGestures(onDoubleTap = {
                        scale = 1f
                        offset = Offset.Zero
                    })
                }
        )
    }
}
```

---

## ロック可能なトランスフォーム

```kotlin
@Composable
fun LockableTransform() {
    var scale by remember { mutableFloatStateOf(1f) }
    var rotation by remember { mutableFloatStateOf(0f) }
    var lockRotation by remember { mutableStateOf(false) }
    var lockScale by remember { mutableStateOf(false) }

    val state = rememberTransformableState { zoomChange, _, rotationChange ->
        if (!lockScale) scale = (scale * zoomChange).coerceIn(0.5f, 3f)
        if (!lockRotation) rotation += rotationChange
    }

    Column(Modifier.fillMaxSize()) {
        Row(Modifier.padding(16.dp), horizontalArrangement = Arrangement.spacedBy(16.dp)) {
            FilterChip(
                selected = lockRotation,
                onClick = { lockRotation = !lockRotation },
                label = { Text("回転ロック") }
            )
            FilterChip(
                selected = lockScale,
                onClick = { lockScale = !lockScale },
                label = { Text("拡大ロック") }
            )
        }

        Box(
            Modifier.weight(1f).fillMaxWidth()
                .transformable(state)
                .graphicsLayer { scaleX = scale; scaleY = scale; rotationZ = rotation },
            contentAlignment = Alignment.Center
        ) {
            Card(Modifier.size(150.dp)) {
                Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                    Text("操作対象")
                }
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `transformable` | トランスフォーム検出 |
| `rememberTransformableState` | 状態管理 |
| `graphicsLayer` | 変換適用 |
| `coerceIn` | 範囲制限 |

- `rememberTransformableState`でズーム/回転/パンの変化量を取得
- `graphicsLayer`で`scaleX/Y`、`rotationZ`、`translationX/Y`を適用
- `coerceIn`でズーム範囲を制限
- ダブルタップリセットと組み合わせて画像ビューアーに

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose GestureDetector](https://zenn.dev/myougatheaxo/articles/android-compose-compose-gesture-detector-2026)
- [Compose PointerInput](https://zenn.dev/myougatheaxo/articles/android-compose-compose-pointer-input-2026)
- [Compose DragDrop](https://zenn.dev/myougatheaxo/articles/android-compose-compose-drag-drop-2026)
