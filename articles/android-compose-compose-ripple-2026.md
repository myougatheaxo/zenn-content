---
title: "Ripple/Indication完全ガイド — カスタムリップル/無効化/バウンスエフェクト"
emoji: "💧"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**Ripple/Indication**（カスタムリップル、リップル無効化、バウンスエフェクト、カスタムIndication）を解説します。

---

## リップル制御

```kotlin
@Composable
fun RippleExamples() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        // デフォルトリップル
        Box(
            Modifier
                .size(100.dp)
                .background(MaterialTheme.colorScheme.primaryContainer)
                .clickable { }
        )

        // カスタムリップル色
        Box(
            Modifier
                .size(100.dp)
                .background(MaterialTheme.colorScheme.secondaryContainer)
                .clickable(
                    interactionSource = remember { MutableInteractionSource() },
                    indication = ripple(color = Color.Red)
                ) { }
        )

        // リップル無効化
        Box(
            Modifier
                .size(100.dp)
                .background(MaterialTheme.colorScheme.tertiaryContainer)
                .clickable(
                    interactionSource = remember { MutableInteractionSource() },
                    indication = null
                ) { }
        )

        // 有界リップル
        Box(
            Modifier
                .size(100.dp)
                .background(Color.LightGray)
                .clickable(
                    interactionSource = remember { MutableInteractionSource() },
                    indication = ripple(bounded = false, radius = 80.dp)
                ) { },
            contentAlignment = Alignment.Center
        ) { Text("無界") }
    }
}
```

---

## カスタムIndication

```kotlin
class ScaleIndication : IndicationNodeFactory {
    override fun create(interactionSource: InteractionSource): DelegatableNode {
        return ScaleIndicationNode(interactionSource)
    }
}

private class ScaleIndicationNode(
    private val interactionSource: InteractionSource
) : Modifier.Node(), DrawModifierNode {
    private var isPressed = false

    override fun ContentDrawScope.draw() {
        val scale = if (isPressed) 0.95f else 1f
        scale(scale) {
            this@draw.drawContent()
        }
    }

    override fun onAttach() {
        coroutineScope.launch {
            interactionSource.interactions.collect { interaction ->
                isPressed = interaction is PressInteraction.Press
            }
        }
    }
}

// 使用
@Composable
fun ScaleButton(onClick: () -> Unit, content: @Composable () -> Unit) {
    Box(
        Modifier.clickable(
            interactionSource = remember { MutableInteractionSource() },
            indication = ScaleIndication()
        ) { onClick() }
    ) { content() }
}
```

---

## InteractionSource活用

```kotlin
@Composable
fun InteractiveCard(onClick: () -> Unit) {
    val interactionSource = remember { MutableInteractionSource() }
    val isPressed by interactionSource.collectIsPressedAsState()
    val isHovered by interactionSource.collectIsHoveredAsState()

    val elevation by animateDpAsState(
        targetValue = when {
            isPressed -> 1.dp
            isHovered -> 8.dp
            else -> 4.dp
        }, label = "elevation"
    )

    Card(
        modifier = Modifier
            .fillMaxWidth()
            .clickable(interactionSource = interactionSource, indication = ripple()) { onClick() },
        elevation = CardDefaults.cardElevation(defaultElevation = elevation)
    ) {
        Text("インタラクティブカード", Modifier.padding(16.dp))
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `ripple()` | Material リップル |
| `indication = null` | リップル無効化 |
| `IndicationNodeFactory` | カスタムエフェクト |
| `InteractionSource` | 状態監視 |

- `ripple()`でカスタムカラー/サイズのリップル
- `indication = null`でリップル完全無効化
- `IndicationNodeFactory`でスケール/バウンスエフェクト
- `InteractionSource`でプレス/ホバー状態を監視

---

8種類のAndroidアプリテンプレート（UIエフェクト対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [アニメーション](https://zenn.dev/myougatheaxo/articles/android-compose-animation-spring-2026)
- [ジェスチャー](https://zenn.dev/myougatheaxo/articles/android-compose-compose-gesture-2026)
- [カスタムShape](https://zenn.dev/myougatheaxo/articles/android-compose-compose-shape-2026)
