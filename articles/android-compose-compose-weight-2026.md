---
title: "Compose Weight完全ガイド — Modifier.weight/比率レイアウト/動的分割"
emoji: "⚖️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "layout"]
published: true
---

## この記事で学べること

**Compose Weight**（Modifier.weight、比率レイアウト、動的分割、fill設定）を解説します。

---

## 基本weight

```kotlin
@Composable
fun WeightDemo() {
    Row(Modifier.fillMaxWidth().height(50.dp)) {
        // 1:2:1の比率
        Box(Modifier.weight(1f).fillMaxHeight().background(Color.Red))
        Box(Modifier.weight(2f).fillMaxHeight().background(Color.Green))
        Box(Modifier.weight(1f).fillMaxHeight().background(Color.Blue))
    }
}
```

---

## 固定幅+残りスペース

```kotlin
@Composable
fun FixedPlusWeightLayout() {
    Row(Modifier.fillMaxWidth().padding(16.dp), verticalAlignment = Alignment.CenterVertically) {
        // 固定幅アイコン
        Icon(Icons.Default.Person, null, Modifier.size(40.dp))
        Spacer(Modifier.width(12.dp))

        // 残りスペースをテキストが使用
        Column(Modifier.weight(1f)) {
            Text("ユーザー名", style = MaterialTheme.typography.titleMedium)
            Text("ステータスメッセージ", style = MaterialTheme.typography.bodySmall)
        }

        // 固定幅ボタン
        IconButton(onClick = {}) {
            Icon(Icons.Default.MoreVert, "メニュー")
        }
    }
}
```

---

## 動的比率変更

```kotlin
@Composable
fun DynamicRatio() {
    var ratio by remember { mutableFloatStateOf(0.5f) }

    Column(Modifier.padding(16.dp)) {
        Row(Modifier.fillMaxWidth().height(100.dp)) {
            Box(
                Modifier.weight(ratio).fillMaxHeight().background(Color(0xFF2196F3)),
                contentAlignment = Alignment.Center
            ) { Text("${(ratio * 100).toInt()}%", color = Color.White) }

            Box(
                Modifier.weight(1f - ratio).fillMaxHeight().background(Color(0xFFFF9800)),
                contentAlignment = Alignment.Center
            ) { Text("${((1f - ratio) * 100).toInt()}%", color = Color.White) }
        }

        Slider(
            value = ratio,
            onValueChange = { ratio = it.coerceIn(0.1f, 0.9f) },
            modifier = Modifier.padding(top = 16.dp)
        )
    }
}
```

---

## まとめ

| 設定 | 用途 |
|------|------|
| `weight(1f)` | 均等分割 |
| `weight(2f)` | 2倍の幅/高さ |
| 固定+weight | 残りスペース活用 |
| `fill = false` | 最小サイズ維持 |

- `weight`はRow/Column内でのみ使用可能
- 固定サイズ要素と組み合わせて柔軟なレイアウト
- `fill = false`で最小限のスペースに抑制
- 動的比率変更でインタラクティブUI

---

8種類のAndroidアプリテンプレート（レイアウト最適化対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose IntrinsicSize](https://zenn.dev/myougatheaxo/articles/android-compose-compose-intrinsic-size-2026)
- [Compose FlowColumn](https://zenn.dev/myougatheaxo/articles/android-compose-compose-flow-column-2026)
- [Compose LayoutModifier](https://zenn.dev/myougatheaxo/articles/android-compose-compose-layout-modifier-2026)
