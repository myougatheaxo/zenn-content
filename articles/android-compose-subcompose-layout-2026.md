---
title: "SubcomposeLayout完全ガイド — 測定結果に基づく動的レイアウト"
emoji: "📏"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "layout"]
published: true
---

## この記事で学べること

**SubcomposeLayout**（子の測定結果で親を決定、動的サイズ調整、カスタムレイアウト、パフォーマンス注意点）を解説します。

---

## SubcomposeLayoutとは

通常のLayoutは子の測定とコンポジションが同時。SubcomposeLayoutは子の測定結果を使って別の子のコンポジションを制御できます。

```kotlin
@Composable
fun MeasureThenDecide(
    mainContent: @Composable () -> Unit,
    dependentContent: @Composable (IntSize) -> Unit
) {
    SubcomposeLayout { constraints ->
        val mainPlaceables = subcompose("main") { mainContent() }
            .map { it.measure(constraints) }

        val mainSize = IntSize(
            mainPlaceables.maxOf { it.width },
            mainPlaceables.sumOf { it.height }
        )

        val dependentPlaceables = subcompose("dependent") { dependentContent(mainSize) }
            .map { it.measure(constraints) }

        val totalHeight = mainSize.height + dependentPlaceables.sumOf { it.height }

        layout(constraints.maxWidth, totalHeight) {
            var y = 0
            mainPlaceables.forEach { it.placeRelative(0, y); y += it.height }
            dependentPlaceables.forEach { it.placeRelative(0, y); y += it.height }
        }
    }
}
```

---

## 同じ高さのカラム

```kotlin
@Composable
fun EqualHeightColumns(
    leftContent: @Composable () -> Unit,
    rightContent: @Composable () -> Unit
) {
    SubcomposeLayout { constraints ->
        val halfWidth = constraints.maxWidth / 2
        val halfConstraints = constraints.copy(maxWidth = halfWidth)

        val leftPlaceables = subcompose("left") { leftContent() }
            .map { it.measure(halfConstraints) }
        val rightPlaceables = subcompose("right") { rightContent() }
            .map { it.measure(halfConstraints) }

        val maxHeight = maxOf(
            leftPlaceables.maxOfOrNull { it.height } ?: 0,
            rightPlaceables.maxOfOrNull { it.height } ?: 0
        )

        layout(constraints.maxWidth, maxHeight) {
            leftPlaceables.forEach { it.placeRelative(0, 0) }
            rightPlaceables.forEach { it.placeRelative(halfWidth, 0) }
        }
    }
}
```

---

## テキスト幅に合わせたインジケータ

```kotlin
@Composable
fun TextWithMatchingIndicator(text: String, isSelected: Boolean) {
    SubcomposeLayout(Modifier.padding(horizontal = 16.dp)) { constraints ->
        val textPlaceable = subcompose("text") {
            Text(
                text,
                style = MaterialTheme.typography.titleMedium,
                fontWeight = if (isSelected) FontWeight.Bold else FontWeight.Normal
            )
        }.first().measure(constraints)

        val indicatorPlaceable = subcompose("indicator") {
            if (isSelected) {
                Box(
                    Modifier
                        .width(with(LocalDensity.current) { textPlaceable.width.toDp() })
                        .height(3.dp)
                        .background(MaterialTheme.colorScheme.primary, RoundedCornerShape(1.5.dp))
                )
            }
        }.firstOrNull()?.measure(constraints)

        val totalHeight = textPlaceable.height + (indicatorPlaceable?.height ?: 0)

        layout(textPlaceable.width, totalHeight) {
            textPlaceable.placeRelative(0, 0)
            indicatorPlaceable?.placeRelative(0, textPlaceable.height)
        }
    }
}
```

---

## パフォーマンス注意

```kotlin
// ❌ SubcomposeLayoutは通常のLayoutより重い
// 単純なサイズ指定にはModifierを使う

// ✅ Modifierで済む場合
Box(Modifier.fillMaxWidth().height(IntrinsicSize.Min)) {
    Row {
        Column(Modifier.weight(1f).fillMaxHeight()) { /* left */ }
        Column(Modifier.weight(1f).fillMaxHeight()) { /* right */ }
    }
}

// SubcomposeLayoutが必要な場合のみ使用：
// - 子Aの測定結果で子Bのコンポジション内容を変えたい
// - 動的にComposableの数を変えたい
```

---

## まとめ

| 機能 | API |
|------|-----|
| 動的コンポジション | `SubcomposeLayout` |
| 子の測定 | `subcompose().measure()` |
| 配置 | `layout { placeRelative() }` |
| 簡易版 | `IntrinsicSize` |

- `SubcomposeLayout`で測定→コンポジションの2段階制御
- 同じ高さのカラム、テキスト幅連動など
- 通常の`Layout`や`IntrinsicSize`で済む場合はそちら優先
- `subcompose`のslotIdで再コンポジション最適化

---

8種類のAndroidアプリテンプレート（カスタムレイアウト対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [カスタムLayout](https://zenn.dev/myougatheaxo/articles/android-compose-custom-layout-2026)
- [ConstraintLayout](https://zenn.dev/myougatheaxo/articles/android-compose-constraint-layout-compose-2026)
- [Canvas](https://zenn.dev/myougatheaxo/articles/android-compose-canvas-custom-draw-2026)
