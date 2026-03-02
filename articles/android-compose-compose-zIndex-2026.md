---
title: "Compose zIndex完全ガイド — 描画順序/重なり制御/Overlay"
emoji: "📚"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**Compose zIndex**（描画順序制御、要素の重なり、オーバーレイ表現）を解説します。

---

## 基本zIndex

```kotlin
@Composable
fun ZIndexDemo() {
    Box(Modifier.size(200.dp)) {
        // zIndex大きい方が上に描画
        Box(
            Modifier
                .size(100.dp)
                .offset(20.dp, 20.dp)
                .zIndex(1f)
                .background(Color.Red.copy(alpha = 0.8f))
        )
        Box(
            Modifier
                .size(100.dp)
                .offset(60.dp, 60.dp)
                .zIndex(2f)
                .background(Color.Blue.copy(alpha = 0.8f))
        )
        Box(
            Modifier
                .size(100.dp)
                .offset(40.dp, 40.dp)
                .zIndex(3f)
                .background(Color.Green.copy(alpha = 0.8f))
        )
    }
}
```

---

## 選択時に最前面

```kotlin
@Composable
fun SelectableCards(items: List<String>) {
    var selectedIndex by remember { mutableIntStateOf(-1) }

    Box(Modifier.fillMaxWidth().height(200.dp)) {
        items.forEachIndexed { index, item ->
            Card(
                modifier = Modifier
                    .offset(x = (index * 60).dp)
                    .zIndex(if (index == selectedIndex) 10f else index.toFloat())
                    .width(120.dp)
                    .fillMaxHeight()
                    .clickable { selectedIndex = index },
                elevation = CardDefaults.cardElevation(
                    defaultElevation = if (index == selectedIndex) 12.dp else 2.dp
                )
            ) {
                Box(Modifier.fillMaxSize().padding(8.dp)) {
                    Text(item, Modifier.align(Alignment.Center))
                }
            }
        }
    }
}
```

---

## FABオーバーレイ

```kotlin
@Composable
fun OverlayFAB() {
    Box(Modifier.fillMaxSize()) {
        LazyColumn(Modifier.fillMaxSize()) {
            items(50) { Text("アイテム $it", Modifier.padding(16.dp)) }
        }

        // 常に最前面
        FloatingActionButton(
            onClick = {},
            modifier = Modifier
                .align(Alignment.BottomEnd)
                .padding(16.dp)
                .zIndex(Float.MAX_VALUE)
        ) {
            Icon(Icons.Default.Add, "追加")
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `Modifier.zIndex` | 描画順序指定 |
| 高い値 | 前面に描画 |
| 動的zIndex | 選択時に最前面化 |
| `Float.MAX_VALUE` | 常に最前面 |

- `zIndex`は同一親内の兄弟要素の描画順序を制御
- デフォルトはコード順（後に書いたものが上）
- 動的に変更して選択状態を表現
- `elevation`とは別概念（elevationはシャドウ）

---

8種類のAndroidアプリテンプレート（カスタムUI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose GraphicsLayer](https://zenn.dev/myougatheaxo/articles/android-compose-compose-graphicsLayer-2026)
- [Compose Shadow](https://zenn.dev/myougatheaxo/articles/android-compose-compose-shadow-2026)
- [Compose Animation](https://zenn.dev/myougatheaxo/articles/android-compose-compose-animation-2026)
