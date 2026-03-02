---
title: "マーキーテキスト完全ガイド — 自動スクロール/Modifier.marquee/カスタム"
emoji: "📜"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**マーキーテキスト**（Modifier.basicMarquee、カスタムスクロールテキスト、条件付きマーキー）を解説します。

---

## Modifier.basicMarquee

```kotlin
@OptIn(ExperimentalFoundationApi::class)
@Composable
fun MarqueeText(text: String) {
    Text(
        text = text,
        modifier = Modifier
            .fillMaxWidth()
            .basicMarquee(
                iterations = Int.MAX_VALUE,  // 無限ループ
                animationMode = MarqueeAnimationMode.Immediately,
                velocity = 30.dp  // スクロール速度
            ),
        maxLines = 1,
        overflow = TextOverflow.Ellipsis
    )
}
```

---

## 条件付きマーキー

```kotlin
@OptIn(ExperimentalFoundationApi::class)
@Composable
fun ConditionalMarquee(text: String, modifier: Modifier = Modifier) {
    var isOverflowing by remember { mutableStateOf(false) }

    Text(
        text = text,
        modifier = modifier
            .then(
                if (isOverflowing) Modifier.basicMarquee()
                else Modifier
            ),
        maxLines = 1,
        onTextLayout = { result ->
            isOverflowing = result.hasVisualOverflow
        }
    )
}
```

---

## カスタムマーキー

```kotlin
@Composable
fun CustomMarqueeText(
    text: String,
    speed: Float = 50f,
    modifier: Modifier = Modifier
) {
    val textMeasurer = rememberTextMeasurer()
    val textStyle = MaterialTheme.typography.bodyLarge
    val textLayoutResult = remember(text) {
        textMeasurer.measure(text, textStyle)
    }
    val textWidth = textLayoutResult.size.width.toFloat()

    val infiniteTransition = rememberInfiniteTransition(label = "marquee")
    val offset by infiniteTransition.animateFloat(
        initialValue = 0f,
        targetValue = -textWidth,
        animationSpec = infiniteRepeatable(
            animation = tween(
                durationMillis = (textWidth / speed * 1000).toInt(),
                easing = LinearEasing
            ),
            repeatMode = RepeatMode.Restart
        ),
        label = "marqueeOffset"
    )

    Box(modifier.clipToBounds()) {
        Text(
            text = text,
            style = textStyle,
            maxLines = 1,
            softWrap = false,
            modifier = Modifier.offset { IntOffset(offset.toInt(), 0) }
        )
        // 2つ目のテキスト（シームレスループ用）
        Text(
            text = text,
            style = textStyle,
            maxLines = 1,
            softWrap = false,
            modifier = Modifier.offset { IntOffset((offset + textWidth + 100).toInt(), 0) }
        )
    }
}
```

---

## 音楽プレイヤー風

```kotlin
@OptIn(ExperimentalFoundationApi::class)
@Composable
fun NowPlayingBar(title: String, artist: String) {
    Surface(
        Modifier.fillMaxWidth(),
        color = MaterialTheme.colorScheme.surfaceVariant
    ) {
        Row(
            Modifier.padding(12.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            Box(
                Modifier.size(48.dp).background(
                    MaterialTheme.colorScheme.primary, RoundedCornerShape(8.dp)
                ),
                contentAlignment = Alignment.Center
            ) {
                Icon(Icons.Default.MusicNote, null, tint = MaterialTheme.colorScheme.onPrimary)
            }

            Spacer(Modifier.width(12.dp))

            Column(Modifier.weight(1f)) {
                Text(
                    title,
                    modifier = Modifier.basicMarquee(),
                    maxLines = 1,
                    style = MaterialTheme.typography.titleSmall
                )
                Text(artist, maxLines = 1, style = MaterialTheme.typography.bodySmall)
            }

            IconButton(onClick = { }) { Icon(Icons.Default.PlayArrow, "再生") }
        }
    }
}
```

---

## まとめ

| 手法 | 特徴 |
|------|------|
| `basicMarquee` | 公式API、簡単 |
| 条件付き | オーバーフロー時のみ |
| カスタム | シームレスループ |
| `animateFloat` | 速度制御可能 |

- `Modifier.basicMarquee()`で簡単にマーキー実装
- `onTextLayout`でオーバーフロー検出して条件付き
- カスタム実装でシームレスな無限ループ
- 音楽プレイヤーや通知バーに最適

---

8種類のAndroidアプリテンプレート（リッチUI実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [テキスト計測](https://zenn.dev/myougatheaxo/articles/android-compose-text-measurement-2026)
- [アニメーション](https://zenn.dev/myougatheaxo/articles/android-compose-animation-transition-2026)
- [Material3コンポーネント](https://zenn.dev/myougatheaxo/articles/android-compose-material3-components-2026)
