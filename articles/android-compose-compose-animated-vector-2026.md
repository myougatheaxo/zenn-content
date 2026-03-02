---
title: "Compose AnimatedVectorDrawable完全ガイド — AVD/アニメーションアイコン/状態遷移"
emoji: "🎭"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "animation"]
published: true
---

## この記事で学べること

**Compose AnimatedVectorDrawable**（AVDアニメーション、アイコン状態遷移、XML→Compose変換）を解説します。

---

## AnimatedVectorDrawable基本

```kotlin
@Composable
fun AnimatedIconDemo() {
    var atEnd by remember { mutableStateOf(false) }
    val painter = rememberAnimatedVectorPainter(
        animatedImageVector = AnimatedImageVector.animatedVectorResource(
            R.drawable.avd_play_to_pause
        ),
        atEnd = atEnd
    )

    Image(
        painter = painter,
        contentDescription = "再生/一時停止",
        modifier = Modifier
            .size(48.dp)
            .clickable { atEnd = !atEnd }
    )
}
```

---

## トグルアイコン

```kotlin
@Composable
fun ToggleIcon(
    isChecked: Boolean,
    onToggle: () -> Unit
) {
    val painter = rememberAnimatedVectorPainter(
        animatedImageVector = AnimatedImageVector.animatedVectorResource(
            R.drawable.avd_checkbox
        ),
        atEnd = isChecked
    )

    IconButton(onClick = onToggle) {
        Image(
            painter = painter,
            contentDescription = if (isChecked) "チェック済み" else "未チェック",
            modifier = Modifier.size(24.dp),
            colorFilter = ColorFilter.tint(MaterialTheme.colorScheme.primary)
        )
    }
}

@Composable
fun ToggleDemo() {
    var checked by remember { mutableStateOf(false) }
    Row(verticalAlignment = Alignment.CenterVertically) {
        ToggleIcon(isChecked = checked, onToggle = { checked = !checked })
        Text(if (checked) "完了" else "未完了")
    }
}
```

---

## Compose代替アニメーション

```kotlin
@Composable
fun ComposeAnimatedIcon(isPlaying: Boolean) {
    val transition = updateTransition(targetState = isPlaying, label = "play")

    val rotation by transition.animateFloat(label = "rotation") { playing ->
        if (playing) 0f else 90f
    }
    val scale by transition.animateFloat(label = "scale") { playing ->
        if (playing) 1f else 0.8f
    }

    Icon(
        imageVector = if (isPlaying) Icons.Default.Pause else Icons.Default.PlayArrow,
        contentDescription = null,
        modifier = Modifier
            .size(48.dp)
            .graphicsLayer {
                rotationZ = rotation
                scaleX = scale
                scaleY = scale
            }
    )
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `AnimatedImageVector` | AVDリソース読み込み |
| `rememberAnimatedVectorPainter` | アニメーションPainter |
| `atEnd` | アニメーション状態 |
| `updateTransition` | Compose代替 |

- `AnimatedImageVector`でXML AVDをCompose化
- `atEnd`トグルでアニメーション再生
- `ColorFilter.tint`でアイコン色変更
- Compose APIで代替アニメーションも実装可能

---

8種類のAndroidアプリテンプレート（アニメーション対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose ImageVector](https://zenn.dev/myougatheaxo/articles/android-compose-compose-image-vector-2026)
- [Compose Canvas](https://zenn.dev/myougatheaxo/articles/android-compose-compose-canvas-2026)
- [Compose Animation](https://zenn.dev/myougatheaxo/articles/android-compose-compose-animation-2026)
