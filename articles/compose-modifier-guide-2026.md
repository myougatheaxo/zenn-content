---
title: "Modifier完全ガイド — Composeの装飾・レイアウト・イベントを使いこなす"
emoji: "🧩"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

ComposeのUI構築で最も重要な**Modifier**の使い方を体系的に解説します。

---

## Modifierの基本

```kotlin
@Composable
fun ModifierBasics() {
    Text(
        "Hello",
        modifier = Modifier
            .fillMaxWidth()
            .padding(16.dp)
            .background(Color.LightGray, RoundedCornerShape(8.dp))
            .padding(8.dp) // 内側のpadding
    )
}
```

**順序が重要**: Modifierはチェーン順に適用される。

```kotlin
// 背景の外側にpadding → 背景が小さくなる
Modifier.padding(16.dp).background(Color.Red)

// 背景の内側にpadding → 背景が大きくなる
Modifier.background(Color.Red).padding(16.dp)
```

---

## サイズ指定

```kotlin
@Composable
fun SizeModifiers() {
    Column {
        // 固定サイズ
        Box(Modifier.size(100.dp).background(Color.Red))

        // 幅・高さ個別
        Box(Modifier.width(200.dp).height(50.dp).background(Color.Blue))

        // 親の割合
        Box(Modifier.fillMaxWidth(0.8f).height(40.dp).background(Color.Green))

        // 最小/最大
        Box(
            Modifier
                .widthIn(min = 100.dp, max = 300.dp)
                .heightIn(min = 50.dp)
                .background(Color.Yellow)
        ) {
            Text("可変幅")
        }

        // アスペクト比
        Box(
            Modifier
                .fillMaxWidth()
                .aspectRatio(16f / 9f)
                .background(Color.Cyan)
        )
    }
}
```

---

## クリック・タッチ

```kotlin
@Composable
fun ClickModifiers() {
    // 基本クリック
    Box(
        Modifier
            .size(100.dp)
            .clickable { /* クリック処理 */ }
    )

    // リップルなしクリック
    Box(
        Modifier
            .size(100.dp)
            .clickable(
                indication = null,
                interactionSource = remember { MutableInteractionSource() }
            ) { /* 処理 */ }
    )

    // 長押し対応
    Box(
        Modifier
            .size(100.dp)
            .combinedClickable(
                onClick = { /* タップ */ },
                onLongClick = { /* 長押し */ },
                onDoubleClick = { /* ダブルタップ */ }
            )
    )
}
```

---

## スクロール

```kotlin
@Composable
fun ScrollModifiers() {
    // 縦スクロール
    Column(
        Modifier
            .fillMaxSize()
            .verticalScroll(rememberScrollState())
    ) {
        repeat(50) {
            Text("Item $it", Modifier.padding(8.dp))
        }
    }

    // 横スクロール
    Row(
        Modifier.horizontalScroll(rememberScrollState())
    ) {
        repeat(20) {
            Box(Modifier.size(80.dp).background(Color.Gray).padding(4.dp))
        }
    }
}
```

---

## 描画系Modifier

```kotlin
@Composable
fun DrawModifiers() {
    // 角丸クリップ
    Image(
        painter = painterResource(R.drawable.photo),
        contentDescription = null,
        modifier = Modifier
            .size(120.dp)
            .clip(RoundedCornerShape(16.dp)),
        contentScale = ContentScale.Crop
    )

    // 円形クリップ
    Image(
        painter = painterResource(R.drawable.avatar),
        contentDescription = null,
        modifier = Modifier
            .size(80.dp)
            .clip(CircleShape)
    )

    // ボーダー
    Box(
        Modifier
            .size(100.dp)
            .border(2.dp, Color.Red, RoundedCornerShape(8.dp))
    )

    // 影
    Box(
        Modifier
            .size(100.dp)
            .shadow(8.dp, RoundedCornerShape(12.dp))
            .background(Color.White)
    )

    // 透明度
    Box(
        Modifier
            .size(100.dp)
            .alpha(0.5f)
            .background(Color.Blue)
    )

    // 回転
    Box(
        Modifier
            .size(100.dp)
            .rotate(45f)
            .background(Color.Green)
    )
}
```

---

## カスタムModifier

```kotlin
// 拡張関数で作成
fun Modifier.cardStyle(): Modifier = this
    .fillMaxWidth()
    .shadow(4.dp, RoundedCornerShape(12.dp))
    .background(Color.White, RoundedCornerShape(12.dp))
    .padding(16.dp)

// composed で状態を持つModifier
fun Modifier.shimmerEffect(): Modifier = composed {
    val transition = rememberInfiniteTransition(label = "shimmer")
    val alpha by transition.animateFloat(
        initialValue = 0.3f,
        targetValue = 1f,
        animationSpec = infiniteRepeatable(
            animation = tween(1000),
            repeatMode = RepeatMode.Reverse
        ),
        label = "alpha"
    )
    this.alpha(alpha)
}

// 使用例
@Composable
fun StyledCard() {
    Box(Modifier.cardStyle()) {
        Text("カードコンテンツ")
    }
}
```

---

## レイアウト系Modifier

```kotlin
@Composable
fun LayoutModifiers() {
    Box(Modifier.fillMaxSize()) {
        // 位置指定
        Text("左上", Modifier.align(Alignment.TopStart))
        Text("中央", Modifier.align(Alignment.Center))
        Text("右下", Modifier.align(Alignment.BottomEnd))

        // オフセット
        Text(
            "オフセット",
            Modifier.offset(x = 20.dp, y = 30.dp)
        )
    }

    // weight（Row/Column内）
    Row(Modifier.fillMaxWidth()) {
        Box(Modifier.weight(1f).height(50.dp).background(Color.Red))
        Box(Modifier.weight(2f).height(50.dp).background(Color.Blue))
    }
}
```

---

## まとめ

- **順序が重要**: padding → background と background → padding で結果が変わる
- **サイズ**: `size`, `fillMaxWidth`, `widthIn`, `aspectRatio`
- **クリック**: `clickable`, `combinedClickable`（長押し/ダブルタップ）
- **描画**: `clip`, `border`, `shadow`, `alpha`, `rotate`
- **レイアウト**: `align`, `offset`, `weight`
- **カスタム**: 拡張関数 or `composed`で再利用可能なModifier

---

8種類のAndroidアプリテンプレート（Modifier活用設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Composeアニメーション入門](https://zenn.dev/myougatheaxo/articles/compose-animation-basics-2026)
- [Canvas完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-canvas-drawing-2026)
- [ジェスチャー入門](https://zenn.dev/myougatheaxo/articles/compose-gesture-touch-2026)
