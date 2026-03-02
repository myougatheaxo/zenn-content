---
title: "FAB完全ガイド — ComposeでFloatingActionButtonを実装"
emoji: "➕"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

Material3の**FloatingActionButton**の全バリエーションと実装パターンを解説します。

---

## 基本のFAB

```kotlin
@Composable
fun BasicFab() {
    Scaffold(
        floatingActionButton = {
            FloatingActionButton(onClick = { /* 追加 */ }) {
                Icon(Icons.Default.Add, "追加")
            }
        }
    ) { padding ->
        // コンテンツ
    }
}
```

---

## SmallFloatingActionButton

```kotlin
SmallFloatingActionButton(onClick = { /* ... */ }) {
    Icon(Icons.Default.Add, "追加", Modifier.size(24.dp))
}
```

---

## LargeFloatingActionButton

```kotlin
LargeFloatingActionButton(onClick = { /* ... */ }) {
    Icon(Icons.Default.Add, "追加", Modifier.size(36.dp))
}
```

---

## ExtendedFloatingActionButton

```kotlin
ExtendedFloatingActionButton(
    onClick = { /* 新規作成 */ },
    icon = { Icon(Icons.Default.Edit, null) },
    text = { Text("新規作成") }
)
```

---

## スクロールで縮小するExtended FAB

```kotlin
@Composable
fun CollapsibleFab() {
    val listState = rememberLazyListState()
    val isExpanded by remember {
        derivedStateOf { listState.firstVisibleItemIndex == 0 }
    }

    Scaffold(
        floatingActionButton = {
            ExtendedFloatingActionButton(
                onClick = { /* ... */ },
                expanded = isExpanded,
                icon = { Icon(Icons.Default.Add, null) },
                text = { Text("新規追加") }
            )
        }
    ) { padding ->
        LazyColumn(state = listState, contentPadding = padding) {
            items(100) { Text("アイテム $it", Modifier.padding(16.dp)) }
        }
    }
}
```

---

## FABメニュー（展開式）

```kotlin
@Composable
fun ExpandableFab() {
    var expanded by remember { mutableStateOf(false) }

    Column(horizontalAlignment = Alignment.End) {
        AnimatedVisibility(visible = expanded) {
            Column(
                horizontalAlignment = Alignment.End,
                verticalArrangement = Arrangement.spacedBy(12.dp)
            ) {
                Row(verticalAlignment = Alignment.CenterVertically) {
                    Surface(
                        shape = RoundedCornerShape(4.dp),
                        tonalElevation = 2.dp
                    ) {
                        Text("写真", Modifier.padding(horizontal = 8.dp, vertical = 4.dp))
                    }
                    Spacer(Modifier.width(8.dp))
                    SmallFloatingActionButton(onClick = { expanded = false }) {
                        Icon(Icons.Default.CameraAlt, "写真")
                    }
                }
                Row(verticalAlignment = Alignment.CenterVertically) {
                    Surface(
                        shape = RoundedCornerShape(4.dp),
                        tonalElevation = 2.dp
                    ) {
                        Text("メモ", Modifier.padding(horizontal = 8.dp, vertical = 4.dp))
                    }
                    Spacer(Modifier.width(8.dp))
                    SmallFloatingActionButton(onClick = { expanded = false }) {
                        Icon(Icons.Default.Note, "メモ")
                    }
                }

                Spacer(Modifier.height(4.dp))
            }
        }

        FloatingActionButton(
            onClick = { expanded = !expanded }
        ) {
            Icon(
                Icons.Default.Add,
                "メニュー",
                modifier = Modifier.graphicsLayer {
                    rotationZ = if (expanded) 45f else 0f
                }
            )
        }
    }
}
```

---

## カスタムカラー

```kotlin
FloatingActionButton(
    onClick = { /* ... */ },
    containerColor = MaterialTheme.colorScheme.tertiary,
    contentColor = MaterialTheme.colorScheme.onTertiary
) {
    Icon(Icons.Default.Add, "追加")
}
```

---

## まとめ

- 4種類: `FAB` / `SmallFAB` / `LargeFAB` / `ExtendedFAB`
- `Scaffold.floatingActionButton`で配置
- `ExtendedFAB`の`expanded`でスクロール連動
- `AnimatedVisibility`で展開式FABメニュー
- `derivedStateOf`でスクロール状態を効率的に監視

---

8種類のAndroidアプリテンプレート（FAB設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Material3コンポーネントカタログ](https://zenn.dev/myougatheaxo/articles/compose-material3-components-2026)
- [Composeアニメーション入門](https://zenn.dev/myougatheaxo/articles/compose-animation-basics-2026)
- [BottomSheet完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-bottom-sheet-2026)
