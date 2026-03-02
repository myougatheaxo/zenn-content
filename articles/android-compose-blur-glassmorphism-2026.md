---
title: "ブラー/グラスモーフィズム完全ガイド — RenderEffect/半透明/背景ぼかし"
emoji: "🪟"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "design"]
published: true
---

## この記事で学べること

**ブラー/グラスモーフィズム**（RenderEffect、背景ぼかし、半透明カード、フロストガラス効果）を解説します。

---

## Modifier.blur

```kotlin
@Composable
fun BlurExample() {
    var blurRadius by remember { mutableFloatStateOf(0f) }

    Column(Modifier.padding(16.dp)) {
        Slider(
            value = blurRadius,
            onValueChange = { blurRadius = it },
            valueRange = 0f..20f
        )

        Image(
            painter = painterResource(R.drawable.photo),
            contentDescription = null,
            modifier = Modifier
                .fillMaxWidth()
                .height(200.dp)
                .clip(RoundedCornerShape(16.dp))
                .blur(blurRadius.dp)
        )
    }
}
```

---

## グラスモーフィズムカード

```kotlin
@Composable
fun GlassCard(
    modifier: Modifier = Modifier,
    content: @Composable () -> Unit
) {
    Box(
        modifier = modifier
            .clip(RoundedCornerShape(20.dp))
            .background(
                Brush.linearGradient(
                    colors = listOf(
                        Color.White.copy(alpha = 0.25f),
                        Color.White.copy(alpha = 0.1f)
                    )
                )
            )
            .border(
                width = 1.dp,
                brush = Brush.linearGradient(
                    colors = listOf(
                        Color.White.copy(alpha = 0.4f),
                        Color.White.copy(alpha = 0.1f)
                    )
                ),
                shape = RoundedCornerShape(20.dp)
            )
            .padding(20.dp)
    ) {
        content()
    }
}

@Composable
fun GlassCardDemo() {
    Box(Modifier.fillMaxSize()) {
        // 背景画像
        Image(
            painter = painterResource(R.drawable.background),
            contentDescription = null,
            modifier = Modifier.fillMaxSize(),
            contentScale = ContentScale.Crop
        )

        // グラスカード
        GlassCard(
            modifier = Modifier
                .align(Alignment.Center)
                .width(300.dp)
        ) {
            Column {
                Text("Glass Card", color = Color.White, fontSize = 24.sp, fontWeight = FontWeight.Bold)
                Spacer(Modifier.height(8.dp))
                Text("半透明のフロストガラス効果", color = Color.White.copy(alpha = 0.8f))
            }
        }
    }
}
```

---

## RenderEffect（API 31+）

```kotlin
@RequiresApi(Build.VERSION_CODES.S)
@Composable
fun RenderEffectBlur() {
    Box(
        Modifier
            .fillMaxWidth()
            .height(200.dp)
            .graphicsLayer {
                renderEffect = RenderEffect
                    .createBlurEffect(25f, 25f, Shader.TileMode.CLAMP)
                    .asComposeRenderEffect()
            }
    ) {
        Image(
            painter = painterResource(R.drawable.photo),
            contentDescription = null,
            modifier = Modifier.fillMaxSize(),
            contentScale = ContentScale.Crop
        )
    }
}
```

---

## ぼかし付きBottomBar

```kotlin
@Composable
fun BlurredBottomBar(
    items: List<BottomNavItem>,
    selectedIndex: Int,
    onSelect: (Int) -> Unit
) {
    Box(
        Modifier
            .fillMaxWidth()
            .height(80.dp)
            .clip(RoundedCornerShape(topStart = 20.dp, topEnd = 20.dp))
            .background(MaterialTheme.colorScheme.surface.copy(alpha = 0.7f))
            .blur(10.dp)
    ) {
        Row(
            Modifier.fillMaxSize().padding(horizontal = 16.dp),
            horizontalArrangement = Arrangement.SpaceEvenly,
            verticalAlignment = Alignment.CenterVertically
        ) {
            items.forEachIndexed { index, item ->
                IconButton(onClick = { onSelect(index) }) {
                    Icon(
                        imageVector = item.icon,
                        contentDescription = item.label,
                        tint = if (index == selectedIndex)
                            MaterialTheme.colorScheme.primary
                        else MaterialTheme.colorScheme.onSurfaceVariant
                    )
                }
            }
        }
    }
}
```

---

## まとめ

| 手法 | API Level | 用途 |
|------|-----------|------|
| `Modifier.blur` | 全バージョン | 基本ぼかし |
| `RenderEffect` | API 31+ | 高性能ぼかし |
| 半透明Background | 全バージョン | グラス効果 |
| Border + Alpha | 全バージョン | フロスト境界 |

- `Modifier.blur`で簡単にぼかし効果
- 半透明 + ボーダーでグラスモーフィズム
- `RenderEffect`で高パフォーマンスなぼかし（API 31+）
- パフォーマンス: ぼかし半径は必要最小限に

---

8種類のAndroidアプリテンプレート（モダンUI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Canvas描画](https://zenn.dev/myougatheaxo/articles/android-compose-canvas-custom-drawing-2026)
- [Material3テーマ](https://zenn.dev/myougatheaxo/articles/android-compose-material3-theme-2026)
- [アニメーション](https://zenn.dev/myougatheaxo/articles/android-compose-animation-transition-2026)
