---
title: "FAB完全ガイド — FloatingActionButton/ExtendedFAB/SmallFAB/LargeFAB"
emoji: "➕"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**FloatingActionButton**（FAB、ExtendedFAB、SmallFAB、LargeFAB、アニメーション）を解説します。

---

## FABの種類

```kotlin
@Composable
fun FabShowcase() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(16.dp)) {
        // 標準FAB
        FloatingActionButton(onClick = {}) {
            Icon(Icons.Default.Add, "追加")
        }

        // Small FAB
        SmallFloatingActionButton(onClick = {}) {
            Icon(Icons.Default.Edit, "編集")
        }

        // Large FAB
        LargeFloatingActionButton(onClick = {}) {
            Icon(Icons.Default.Add, "追加", modifier = Modifier.size(36.dp))
        }

        // Extended FAB
        ExtendedFloatingActionButton(
            onClick = {},
            icon = { Icon(Icons.Default.Create, null) },
            text = { Text("新規作成") }
        )
    }
}
```

---

## スクロール連動FAB

```kotlin
@Composable
fun ScrollAwareFab() {
    val listState = rememberLazyListState()
    val isExpanded by remember {
        derivedStateOf { listState.firstVisibleItemIndex == 0 }
    }

    Scaffold(
        floatingActionButton = {
            ExtendedFloatingActionButton(
                onClick = {},
                expanded = isExpanded,
                icon = { Icon(Icons.Default.Add, null) },
                text = { Text("追加") }
            )
        }
    ) { padding ->
        LazyColumn(state = listState, modifier = Modifier.padding(padding)) {
            items(50) { ListItem(headlineContent = { Text("Item $it") }) }
        }
    }
}
```

---

## Speed Dial FAB

```kotlin
@Composable
fun SpeedDialFab() {
    var expanded by remember { mutableStateOf(false) }

    Box(Modifier.fillMaxSize().padding(16.dp), contentAlignment = Alignment.BottomEnd) {
        Column(horizontalAlignment = Alignment.End, verticalArrangement = Arrangement.spacedBy(12.dp)) {
            AnimatedVisibility(visible = expanded, enter = fadeIn() + scaleIn(), exit = fadeOut() + scaleOut()) {
                Column(horizontalAlignment = Alignment.End, verticalArrangement = Arrangement.spacedBy(12.dp)) {
                    SmallFloatingActionButton(onClick = {}) { Icon(Icons.Default.Photo, "写真") }
                    SmallFloatingActionButton(onClick = {}) { Icon(Icons.Default.CameraAlt, "カメラ") }
                    SmallFloatingActionButton(onClick = {}) { Icon(Icons.Default.AttachFile, "ファイル") }
                }
            }

            FloatingActionButton(
                onClick = { expanded = !expanded },
                containerColor = if (expanded) MaterialTheme.colorScheme.error
                    else MaterialTheme.colorScheme.primaryContainer
            ) {
                Icon(
                    if (expanded) Icons.Default.Close else Icons.Default.Add,
                    "メニュー"
                )
            }
        }
    }
}
```

---

## まとめ

| FAB | 用途 |
|-----|------|
| `FloatingActionButton` | 標準FAB |
| `SmallFloatingActionButton` | 小型FAB |
| `LargeFloatingActionButton` | 大型FAB |
| `ExtendedFloatingActionButton` | テキスト付きFAB |

- 4種類のFABをアクションに応じて使い分け
- `ExtendedFAB`のexpandedでアイコンのみ/テキスト付き切替
- Speed Dialパターンで複数アクション表示
- `Scaffold`のfloatingActionButtonで配置

---

8種類のAndroidアプリテンプレート（Material3 UI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Scaffold](https://zenn.dev/myougatheaxo/articles/android-compose-compose-scaffold-2026)
- [BottomAppBar](https://zenn.dev/myougatheaxo/articles/android-compose-compose-bottom-app-bar-2026)
- [AnimatedVisibility](https://zenn.dev/myougatheaxo/articles/android-compose-compose-animated-visibility-2026)
