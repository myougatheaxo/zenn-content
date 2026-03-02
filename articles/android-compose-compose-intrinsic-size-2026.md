---
title: "Compose IntrinsicSize完全ガイド — 固有サイズ測定/Min/Max/高さ揃え"
emoji: "📐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "layout"]
published: true
---

## この記事で学べること

**Compose IntrinsicSize**（IntrinsicSize.Min/Max、固有サイズ測定、子要素の高さ/幅揃え）を解説します。

---

## IntrinsicSize.Min

```kotlin
@Composable
fun IntrinsicMinDemo() {
    // 全子要素の高さを最も高いものに揃える
    Row(Modifier.height(IntrinsicSize.Min)) {
        Text(
            "短いテキスト",
            Modifier
                .weight(1f)
                .fillMaxHeight()
                .background(Color(0xFFE3F2FD))
                .padding(8.dp)
        )
        Divider(
            Modifier
                .fillMaxHeight()
                .width(1.dp),
            color = Color.Gray
        )
        Text(
            "長いテキスト\n複数行にわたる\n内容がここに入る",
            Modifier
                .weight(1f)
                .fillMaxHeight()
                .background(Color(0xFFFCE4EC))
                .padding(8.dp)
        )
    }
}
```

---

## テーブルレイアウト

```kotlin
@Composable
fun TableRow(label: String, value: String) {
    Row(
        Modifier
            .fillMaxWidth()
            .height(IntrinsicSize.Min)
    ) {
        Text(
            label,
            Modifier
                .weight(0.4f)
                .fillMaxHeight()
                .background(Color(0xFFF5F5F5))
                .padding(12.dp),
            fontWeight = FontWeight.Bold
        )
        VerticalDivider()
        Text(
            value,
            Modifier
                .weight(0.6f)
                .fillMaxHeight()
                .padding(12.dp)
        )
    }
}

@Composable
fun TableDemo() {
    Column(Modifier.fillMaxWidth()) {
        TableRow("名前", "みょうが")
        HorizontalDivider()
        TableRow("種類", "ウーパールーパー\n（メキシコサラマンダー）")
        HorizontalDivider()
        TableRow("色", "ピンク")
    }
}
```

---

## IntrinsicSize.Max

```kotlin
@Composable
fun IntrinsicMaxDemo() {
    // 最も広い子の幅に合わせる
    Column(Modifier.width(IntrinsicSize.Max)) {
        Button(onClick = {}, Modifier.fillMaxWidth()) { Text("短い") }
        Spacer(Modifier.height(8.dp))
        Button(onClick = {}, Modifier.fillMaxWidth()) { Text("これは長いボタンテキスト") }
        Spacer(Modifier.height(8.dp))
        Button(onClick = {}, Modifier.fillMaxWidth()) { Text("中くらい") }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `IntrinsicSize.Min` | 最小固有サイズに揃え |
| `IntrinsicSize.Max` | 最大固有サイズに揃え |
| `height(IntrinsicSize.Min)` | 行の高さ揃え |
| `width(IntrinsicSize.Max)` | 列の幅揃え |

- `IntrinsicSize.Min`でDivider付きRowの高さを揃える
- `IntrinsicSize.Max`でボタン幅を最大に統一
- 2パス測定のためパフォーマンスに注意
- LazyLayoutでは使用不可

---

8種類のAndroidアプリテンプレート（レイアウト最適化対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose LayoutModifier](https://zenn.dev/myougatheaxo/articles/android-compose-compose-layout-modifier-2026)
- [Compose AspectRatio](https://zenn.dev/myougatheaxo/articles/android-compose-compose-aspect-ratio-2026)
- [Compose AdaptiveLayout](https://zenn.dev/myougatheaxo/articles/android-compose-compose-adaptive-layout-2026)
