---
title: "FAB/SpeedDial完全ガイド — FloatingActionButton/展開メニュー/アニメーション"
emoji: "🎯"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**FAB/SpeedDial**（FloatingActionButton、展開メニュー、アニメーション、ExtendedFAB）を解説します。

---

## 基本FAB

```kotlin
@Composable
fun BasicFabScreen() {
    Scaffold(
        floatingActionButton = {
            FloatingActionButton(
                onClick = { /* アクション */ },
                containerColor = MaterialTheme.colorScheme.primary
            ) {
                Icon(Icons.Default.Add, "追加")
            }
        },
        floatingActionButtonPosition = FabPosition.End
    ) { padding ->
        // コンテンツ
    }
}

// Extended FAB（ラベル付き）
@Composable
fun ExtendedFabExample(listState: LazyListState) {
    val expanded by remember {
        derivedStateOf { listState.firstVisibleItemIndex == 0 }
    }

    ExtendedFloatingActionButton(
        onClick = { /* アクション */ },
        expanded = expanded,
        icon = { Icon(Icons.Default.Edit, null) },
        text = { Text("作成") }
    )
}
```

---

## SpeedDial（展開メニュー）

```kotlin
@Composable
fun SpeedDialFab() {
    var expanded by remember { mutableStateOf(false) }
    val rotation by animateFloatAsState(
        targetValue = if (expanded) 45f else 0f,
        animationSpec = spring(dampingRatio = 0.6f),
        label = "fabRotation"
    )

    Column(horizontalAlignment = Alignment.End) {
        // サブアクション
        AnimatedVisibility(
            visible = expanded,
            enter = fadeIn() + slideInVertically(initialOffsetY = { it }),
            exit = fadeOut() + slideOutVertically(targetOffsetY = { it })
        ) {
            Column(
                horizontalAlignment = Alignment.End,
                verticalArrangement = Arrangement.spacedBy(12.dp),
                modifier = Modifier.padding(bottom = 16.dp)
            ) {
                SpeedDialItem(icon = Icons.Default.CameraAlt, label = "写真") { /* */ }
                SpeedDialItem(icon = Icons.Default.Mic, label = "音声") { /* */ }
                SpeedDialItem(icon = Icons.Default.Description, label = "ファイル") { /* */ }
            }
        }

        // メインFAB
        FloatingActionButton(
            onClick = { expanded = !expanded }
        ) {
            Icon(
                Icons.Default.Add, "メニュー",
                modifier = Modifier.graphicsLayer { rotationZ = rotation }
            )
        }
    }
}

@Composable
fun SpeedDialItem(icon: ImageVector, label: String, onClick: () -> Unit) {
    Row(
        verticalAlignment = Alignment.CenterVertically,
        horizontalArrangement = Arrangement.End
    ) {
        Surface(
            shape = RoundedCornerShape(4.dp),
            shadowElevation = 2.dp
        ) {
            Text(label, Modifier.padding(horizontal = 8.dp, vertical = 4.dp),
                style = MaterialTheme.typography.labelMedium)
        }
        Spacer(Modifier.width(8.dp))
        SmallFloatingActionButton(onClick = onClick) {
            Icon(icon, label)
        }
    }
}
```

---

## スクロール連動FAB

```kotlin
@Composable
fun ScrollAwareFab(listState: LazyListState) {
    val isScrollingUp by remember {
        derivedStateOf {
            listState.firstVisibleItemIndex > 0 ||
                listState.firstVisibleItemScrollOffset > 0
        }
    }

    AnimatedVisibility(
        visible = !isScrollingUp,
        enter = scaleIn(animationSpec = spring()),
        exit = scaleOut(animationSpec = spring())
    ) {
        FloatingActionButton(onClick = { /* */ }) {
            Icon(Icons.Default.Add, "追加")
        }
    }
}
```

---

## まとめ

| 種類 | 用途 |
|------|------|
| `FloatingActionButton` | メインアクション |
| `ExtendedFloatingActionButton` | ラベル付き |
| `SmallFloatingActionButton` | サブアクション |
| SpeedDial | 展開メニュー |

- `FloatingActionButton`でメインアクション
- SpeedDialで複数アクションを展開
- `AnimatedVisibility`でスクロール連動表示/非表示
- `ExtendedFloatingActionButton`でスクロールに応じて伸縮

---

8種類のAndroidアプリテンプレート（FAB実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Material3コンポーネント](https://zenn.dev/myougatheaxo/articles/android-compose-material3-components-2026)
- [アニメーション](https://zenn.dev/myougatheaxo/articles/android-compose-animation-transition-2026)
- [Scaffold](https://zenn.dev/myougatheaxo/articles/android-compose-scaffold-topbar-2026)
