---
title: "Compose PointerInput完全ガイド — マルチタッチ/AwaitPointerEvent/カスタムジェスチャー"
emoji: "✋"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "gesture"]
published: true
---

## この記事で学べること

**Compose PointerInput**（awaitPointerEvent、マルチタッチ、カスタムジェスチャー、ジェスチャー競合解決）を解説します。

---

## awaitPointerEvent

```kotlin
@Composable
fun PointerEventDemo() {
    var pointerInfo by remember { mutableStateOf("タッチしてください") }

    Box(
        Modifier.fillMaxWidth().height(200.dp)
            .background(Color(0xFFE3F2FD), RoundedCornerShape(16.dp))
            .pointerInput(Unit) {
                awaitEachGesture {
                    val down = awaitFirstDown()
                    pointerInfo = "DOWN: (${down.position.x.toInt()}, ${down.position.y.toInt()})"

                    do {
                        val event = awaitPointerEvent()
                        val position = event.changes.first().position
                        pointerInfo = "MOVE: (${position.x.toInt()}, ${position.y.toInt()})"
                    } while (event.changes.any { it.pressed })

                    pointerInfo = "UP"
                }
            },
        contentAlignment = Alignment.Center
    ) { Text(pointerInfo) }
}
```

---

## マルチタッチ

```kotlin
@Composable
fun MultiTouchDemo() {
    var touchCount by remember { mutableIntStateOf(0) }
    var touchPoints by remember { mutableStateOf<List<Offset>>(emptyList()) }

    Box(
        Modifier.fillMaxSize()
            .pointerInput(Unit) {
                awaitEachGesture {
                    awaitFirstDown()
                    do {
                        val event = awaitPointerEvent()
                        touchCount = event.changes.count { it.pressed }
                        touchPoints = event.changes
                            .filter { it.pressed }
                            .map { it.position }
                    } while (event.changes.any { it.pressed })
                    touchCount = 0
                    touchPoints = emptyList()
                }
            }
    ) {
        Canvas(Modifier.fillMaxSize()) {
            touchPoints.forEach { point ->
                drawCircle(Color.Red.copy(alpha = 0.5f), radius = 30f, center = point)
            }
        }
        Text("タッチ数: $touchCount", Modifier.padding(16.dp))
    }
}
```

---

## カスタムジェスチャー

```kotlin
// 3本指タップ検出
suspend fun PointerInputScope.detectThreeFingerTap(onTripleTap: () -> Unit) {
    awaitEachGesture {
        val firstDown = awaitFirstDown()

        // 他の2本の指を待つ
        val secondDown = withTimeoutOrNull(300) { awaitFirstDown(requireUnconsumed = false) }
        val thirdDown = if (secondDown != null) {
            withTimeoutOrNull(300) { awaitFirstDown(requireUnconsumed = false) }
        } else null

        if (secondDown != null && thirdDown != null) {
            // 3本指が揃った
            val event = awaitPointerEvent()
            if (event.changes.none { it.pressed }) {
                onTripleTap()
            }
        }
    }
}

@Composable
fun ThreeFingerDemo() {
    var triggered by remember { mutableStateOf(false) }

    Box(Modifier.fillMaxSize()
        .pointerInput(Unit) { detectThreeFingerTap { triggered = true } },
        contentAlignment = Alignment.Center) {
        Text(if (triggered) "3本指タップ検出!" else "3本指でタップ")
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `awaitPointerEvent` | 低レベルイベント取得 |
| `awaitFirstDown` | 最初のタッチ待ち |
| `awaitEachGesture` | ジェスチャーループ |
| `withTimeoutOrNull` | タイムアウト付き待機 |

- `awaitEachGesture`でジェスチャーのライフサイクルを管理
- `awaitPointerEvent()`で全タッチイベントを取得
- マルチタッチは`event.changes`で全ポインター情報
- カスタムジェスチャーを自由に実装可能

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose GestureDetector](https://zenn.dev/myougatheaxo/articles/android-compose-compose-gesture-detector-2026)
- [Compose Transformable](https://zenn.dev/myougatheaxo/articles/android-compose-compose-transformable-2026)
- [Compose DragDrop](https://zenn.dev/myougatheaxo/articles/android-compose-compose-drag-drop-2026)
