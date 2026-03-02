---
title: "Tooltip完全ガイド — PlainTooltip/RichTooltip/カスタムTooltip"
emoji: "💡"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**Tooltip**（PlainTooltip、RichTooltip、カスタムTooltip、長押しトリガー）を解説します。

---

## PlainTooltip

```kotlin
@Composable
fun PlainTooltipExample() {
    TooltipBox(
        positionProvider = TooltipDefaults.rememberPlainTooltipPositionProvider(),
        tooltip = {
            PlainTooltip { Text("お気に入りに追加") }
        },
        state = rememberTooltipState()
    ) {
        IconButton(onClick = {}) {
            Icon(Icons.Default.Favorite, "お気に入り")
        }
    }
}
```

---

## RichTooltip

```kotlin
@Composable
fun RichTooltipExample() {
    val tooltipState = rememberTooltipState(isPersistent = true)
    val scope = rememberCoroutineScope()

    TooltipBox(
        positionProvider = TooltipDefaults.rememberRichTooltipPositionProvider(),
        tooltip = {
            RichTooltip(
                title = { Text("プレミアム機能") },
                action = {
                    TextButton(onClick = { scope.launch { tooltipState.dismiss() } }) {
                        Text("詳しく見る")
                    }
                }
            ) {
                Text("この機能はプレミアムプランで利用できます。アップグレードして全機能をお使いください。")
            }
        },
        state = tooltipState
    ) {
        IconButton(onClick = {}) {
            Icon(Icons.Default.Star, "プレミアム")
        }
    }
}
```

---

## カスタムTooltip

```kotlin
@Composable
fun CustomTooltip(
    text: String,
    content: @Composable () -> Unit
) {
    var showTooltip by remember { mutableStateOf(false) }

    Box {
        Box(
            Modifier
                .pointerInput(Unit) {
                    detectTapGestures(
                        onLongPress = { showTooltip = true }
                    )
                }
        ) { content() }

        AnimatedVisibility(
            visible = showTooltip,
            enter = fadeIn() + scaleIn(),
            exit = fadeOut() + scaleOut()
        ) {
            Surface(
                color = MaterialTheme.colorScheme.inverseSurface,
                shape = RoundedCornerShape(8.dp),
                modifier = Modifier
                    .offset(y = (-48).dp)
                    .clickable { showTooltip = false }
            ) {
                Text(
                    text,
                    color = MaterialTheme.colorScheme.inverseOnSurface,
                    modifier = Modifier.padding(horizontal = 12.dp, vertical = 6.dp),
                    style = MaterialTheme.typography.bodySmall
                )
            }
        }
    }

    LaunchedEffect(showTooltip) {
        if (showTooltip) {
            delay(3000)
            showTooltip = false
        }
    }
}
```

---

## まとめ

| 種類 | 用途 |
|------|------|
| PlainTooltip | シンプルなヒント |
| RichTooltip | タイトル+説明+アクション |
| カスタム | 独自デザイン |

- `TooltipBox`でMaterial3準拠のTooltip
- `PlainTooltip`でアイコンボタンの説明
- `RichTooltip`で詳細説明+アクションボタン
- `isPersistent = true`でタップまで表示継続

---

8種類のAndroidアプリテンプレート（UI/UX最適化済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [コンテキストメニュー](https://zenn.dev/myougatheaxo/articles/android-compose-context-menu-2026)
- [BottomSheet](https://zenn.dev/myougatheaxo/articles/android-compose-bottomsheet-dialog-2026)
- [アニメーション](https://zenn.dev/myougatheaxo/articles/android-compose-animation-spring-2026)
