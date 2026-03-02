---
title: "レイアウトサイズ完全ガイド — aspectRatio/weight/intrinsicSize/fillMaxSize"
emoji: "📏"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "layout"]
published: true
---

## この記事で学べること

**レイアウトサイズ**（aspectRatio、weight、IntrinsicSize、fillMaxSize系Modifier）を解説します。

---

## aspectRatio

```kotlin
@Composable
fun AspectRatioExamples() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        // 16:9
        Box(
            Modifier
                .fillMaxWidth()
                .aspectRatio(16f / 9f)
                .background(Color.DarkGray, RoundedCornerShape(8.dp)),
            contentAlignment = Alignment.Center
        ) { Text("16:9", color = Color.White) }

        // 1:1（正方形）
        Box(
            Modifier
                .width(150.dp)
                .aspectRatio(1f)
                .background(MaterialTheme.colorScheme.primary, RoundedCornerShape(8.dp)),
            contentAlignment = Alignment.Center
        ) { Text("1:1", color = Color.White) }

        // 4:3
        AsyncImage(
            model = "https://example.com/photo.jpg",
            contentDescription = null,
            modifier = Modifier
                .fillMaxWidth()
                .aspectRatio(4f / 3f)
                .clip(RoundedCornerShape(12.dp)),
            contentScale = ContentScale.Crop
        )
    }
}
```

---

## weight

```kotlin
@Composable
fun WeightExamples() {
    // Row weight
    Row(Modifier.fillMaxWidth().height(50.dp)) {
        Box(Modifier.weight(1f).fillMaxHeight().background(Color.Red))
        Box(Modifier.weight(2f).fillMaxHeight().background(Color.Green))
        Box(Modifier.weight(1f).fillMaxHeight().background(Color.Blue))
    }

    Spacer(Modifier.height(16.dp))

    // Column weight
    Column(Modifier.fillMaxWidth().height(300.dp)) {
        Box(
            Modifier.fillMaxWidth().weight(1f).background(MaterialTheme.colorScheme.primary),
            contentAlignment = Alignment.Center
        ) { Text("ヘッダー", color = Color.White) }

        Box(
            Modifier.fillMaxWidth().weight(3f).background(MaterialTheme.colorScheme.surface),
            contentAlignment = Alignment.Center
        ) { Text("コンテンツ") }

        Box(
            Modifier.fillMaxWidth().weight(1f).background(MaterialTheme.colorScheme.secondary),
            contentAlignment = Alignment.Center
        ) { Text("フッター", color = Color.White) }
    }
}
```

---

## IntrinsicSize

```kotlin
@Composable
fun IntrinsicSizeExample() {
    Row(Modifier.height(IntrinsicSize.Min).padding(16.dp)) {
        Text(
            "短いテキスト",
            modifier = Modifier
                .weight(1f)
                .background(Color.LightGray)
                .padding(8.dp)
        )
        VerticalDivider(
            modifier = Modifier.fillMaxHeight().width(1.dp),
            color = Color.Black
        )
        Text(
            "もっと長いテキストがここにあります。複数行になることもあります。",
            modifier = Modifier
                .weight(1f)
                .background(Color.LightGray)
                .padding(8.dp)
        )
    }
}
```

---

## まとめ

| Modifier | 用途 |
|----------|------|
| `aspectRatio` | アスペクト比固定 |
| `weight` | 比率配分 |
| `IntrinsicSize.Min` | 最小固有サイズ |
| `fillMaxWidth/Height` | 親いっぱいに拡張 |

- `aspectRatio`で画像/動画の比率を維持
- `weight`でRow/Column内の比率配分
- `IntrinsicSize`でDividerの高さを自動調整
- `fillMaxWidth(0.8f)`で80%幅指定も可能

---

8種類のAndroidアプリテンプレート（レイアウト実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ConstraintLayout](https://zenn.dev/myougatheaxo/articles/android-compose-constraint-layout-2026)
- [FlowRow](https://zenn.dev/myougatheaxo/articles/android-compose-compose-flow-row-2026)
- [LazyGrid](https://zenn.dev/myougatheaxo/articles/android-compose-compose-lazy-grid-2026)
