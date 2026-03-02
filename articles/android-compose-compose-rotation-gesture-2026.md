---
title: "Rotation Gesture完全ガイド — 回転ジェスチャー/Transform/マルチタッチ"
emoji: "🔄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "gesture"]
published: true
---

## この記事で学べること

**Rotation Gesture**（回転ジェスチャー、Transform Gesture、ズーム+回転+移動の複合操作）を解説します。

---

## 回転ジェスチャー

```kotlin
@Composable
fun RotatableImage() {
    var rotation by remember { mutableFloatStateOf(0f) }

    Image(
        painter = painterResource(R.drawable.sample),
        contentDescription = "回転可能な画像",
        modifier = Modifier
            .size(200.dp)
            .graphicsLayer { rotationZ = rotation }
            .pointerInput(Unit) {
                detectTransformGestures { _, _, _, rotationChange ->
                    rotation += rotationChange
                }
            }
    )
}
```

---

## ズーム+回転+移動

```kotlin
@Composable
fun TransformableBox() {
    var scale by remember { mutableFloatStateOf(1f) }
    var rotation by remember { mutableFloatStateOf(0f) }
    var offset by remember { mutableStateOf(Offset.Zero) }

    Box(
        Modifier
            .fillMaxSize()
            .pointerInput(Unit) {
                detectTransformGestures { centroid, pan, zoom, rotationDelta ->
                    scale = (scale * zoom).coerceIn(0.5f, 3f)
                    rotation += rotationDelta
                    offset += pan
                }
            },
        contentAlignment = Alignment.Center
    ) {
        Card(
            Modifier
                .size(150.dp)
                .graphicsLayer {
                    scaleX = scale; scaleY = scale
                    rotationZ = rotation
                    translationX = offset.x; translationY = offset.y
                }
        ) {
            Box(Modifier.fillMaxSize().background(Color(0xFF42A5F5)), contentAlignment = Alignment.Center) {
                Text("ドラッグ\n回転\nズーム", textAlign = TextAlign.Center, color = Color.White)
            }
        }
    }
}
```

---

## リセット付きエディタ

```kotlin
@Composable
fun ImageEditor() {
    var scale by remember { mutableFloatStateOf(1f) }
    var rotation by remember { mutableFloatStateOf(0f) }
    var offset by remember { mutableStateOf(Offset.Zero) }

    Column(Modifier.fillMaxSize()) {
        Box(
            Modifier.weight(1f).fillMaxWidth()
                .pointerInput(Unit) {
                    detectTransformGestures { _, pan, zoom, rot ->
                        scale = (scale * zoom).coerceIn(0.3f, 5f)
                        rotation += rot
                        offset += pan
                    }
                },
            contentAlignment = Alignment.Center
        ) {
            Image(
                painter = painterResource(R.drawable.photo),
                contentDescription = null,
                modifier = Modifier.graphicsLayer {
                    scaleX = scale; scaleY = scale
                    rotationZ = rotation
                    translationX = offset.x; translationY = offset.y
                }
            )
        }

        Row(Modifier.fillMaxWidth().padding(8.dp), horizontalArrangement = Arrangement.SpaceEvenly) {
            TextButton(onClick = { rotation -= 90f }) { Text("↺ 90°") }
            TextButton(onClick = { scale = 1f; rotation = 0f; offset = Offset.Zero }) { Text("リセット") }
            TextButton(onClick = { rotation += 90f }) { Text("↻ 90°") }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `detectTransformGestures` | 複合ジェスチャー |
| `rotationZ` | 回転角度 |
| `scaleX/Y` | ズーム |
| `translationX/Y` | 移動 |

- `detectTransformGestures`でズーム/回転/移動を同時検出
- `graphicsLayer`で変換を適用
- `coerceIn`でスケールの範囲制限
- 画像エディタ/地図操作に活用

---

8種類のAndroidアプリテンプレート（ジェスチャー対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [DragDrop](https://zenn.dev/myougatheaxo/articles/android-compose-compose-drag-drop-2026)
- [SwipeToDismiss](https://zenn.dev/myougatheaxo/articles/android-compose-compose-swipe-dismiss-2026)
- [PDF Viewer](https://zenn.dev/myougatheaxo/articles/android-compose-compose-pdf-viewer-2026)
