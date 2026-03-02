---
title: "Clip完全ガイド — clip/CircleShape/RoundedCorner/カスタムクリッピング"
emoji: "✂️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**Clip**（clip Modifier、CircleShape、RoundedCornerShape、カスタムクリッピング）を解説します。

---

## 基本Clip

```kotlin
@Composable
fun ClipExamples() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        // 丸型
        Image(
            painter = painterResource(R.drawable.photo),
            contentDescription = null,
            modifier = Modifier.size(100.dp).clip(CircleShape),
            contentScale = ContentScale.Crop
        )

        // 角丸
        Image(
            painter = painterResource(R.drawable.photo),
            contentDescription = null,
            modifier = Modifier
                .fillMaxWidth()
                .height(200.dp)
                .clip(RoundedCornerShape(16.dp)),
            contentScale = ContentScale.Crop
        )

        // 上だけ角丸
        Box(
            Modifier
                .fillMaxWidth()
                .height(150.dp)
                .clip(RoundedCornerShape(topStart = 24.dp, topEnd = 24.dp))
                .background(MaterialTheme.colorScheme.primaryContainer)
        )

        // カットコーナー
        Box(
            Modifier
                .size(100.dp)
                .clip(CutCornerShape(16.dp))
                .background(MaterialTheme.colorScheme.secondary)
        )
    }
}
```

---

## カスタムShape Clip

```kotlin
val TicketShape = GenericShape { size, _ ->
    val notchRadius = 20f
    moveTo(0f, 0f)
    lineTo(size.width, 0f)
    lineTo(size.width, size.height / 2 - notchRadius)
    arcTo(
        rect = Rect(
            size.width - notchRadius, size.height / 2 - notchRadius,
            size.width + notchRadius, size.height / 2 + notchRadius
        ),
        startAngleDegrees = -90f, sweepAngleDegrees = -180f, forceMoveTo = false
    )
    lineTo(size.width, size.height)
    lineTo(0f, size.height)
    lineTo(0f, size.height / 2 + notchRadius)
    arcTo(
        rect = Rect(
            -notchRadius, size.height / 2 - notchRadius,
            notchRadius, size.height / 2 + notchRadius
        ),
        startAngleDegrees = 90f, sweepAngleDegrees = -180f, forceMoveTo = false
    )
    close()
}

@Composable
fun TicketCard() {
    Box(
        Modifier
            .fillMaxWidth()
            .height(120.dp)
            .clip(TicketShape)
            .background(MaterialTheme.colorScheme.primaryContainer)
            .padding(16.dp)
    ) {
        Text("チケット", style = MaterialTheme.typography.headlineSmall)
    }
}
```

---

## まとめ

| Shape | 用途 |
|-------|------|
| `CircleShape` | 円形クリップ |
| `RoundedCornerShape` | 角丸クリップ |
| `CutCornerShape` | カット角 |
| `GenericShape` | カスタム形状 |

- `clip(CircleShape)`で丸型アバター
- `RoundedCornerShape`で個別角丸指定
- `GenericShape`で自由な形状のクリッピング
- `clip`は`background`/`clickable`の前に適用

---

8種類のAndroidアプリテンプレート（カスタムUI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [カスタムShape](https://zenn.dev/myougatheaxo/articles/android-compose-compose-shape-2026)
- [Border/Shadow](https://zenn.dev/myougatheaxo/articles/android-compose-compose-border-2026)
- [画像読み込み](https://zenn.dev/myougatheaxo/articles/android-compose-compose-image-loading-2026)
