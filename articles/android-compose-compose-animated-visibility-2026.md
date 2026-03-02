---
title: "AnimatedVisibility完全ガイド — Enter/Exit/カスタムアニメーション/条件表示"
emoji: "👁️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "animation"]
published: true
---

## この記事で学べること

**AnimatedVisibility**（EnterTransition、ExitTransition、カスタムアニメーション、条件付き表示）を解説します。

---

## 基本AnimatedVisibility

```kotlin
@Composable
fun AnimatedVisibilityExample() {
    var visible by remember { mutableStateOf(true) }

    Column(Modifier.padding(16.dp)) {
        Button(onClick = { visible = !visible }) {
            Text(if (visible) "非表示" else "表示")
        }

        Spacer(Modifier.height(8.dp))

        // フェード+スライド
        AnimatedVisibility(
            visible = visible,
            enter = fadeIn() + slideInVertically(),
            exit = fadeOut() + slideOutVertically()
        ) {
            Card(Modifier.fillMaxWidth()) {
                Text("アニメーション付きコンテンツ", Modifier.padding(16.dp))
            }
        }
    }
}
```

---

## カスタムEnter/Exit

```kotlin
@Composable
fun CustomTransitionExample() {
    var visible by remember { mutableStateOf(false) }

    Column(Modifier.padding(16.dp)) {
        Button(onClick = { visible = !visible }) { Text("トグル") }

        // 左からスライドイン、右にスライドアウト
        AnimatedVisibility(
            visible = visible,
            enter = slideInHorizontally(
                initialOffsetX = { -it },
                animationSpec = tween(500)
            ) + fadeIn(animationSpec = tween(500)),
            exit = slideOutHorizontally(
                targetOffsetX = { it },
                animationSpec = tween(500)
            ) + fadeOut(animationSpec = tween(500))
        ) {
            Card(Modifier.fillMaxWidth()) {
                Text("左から右へ", Modifier.padding(16.dp))
            }
        }

        Spacer(Modifier.height(8.dp))

        // 拡大/縮小
        AnimatedVisibility(
            visible = visible,
            enter = expandVertically(expandFrom = Alignment.Top) + fadeIn(),
            exit = shrinkVertically(shrinkTowards = Alignment.Top) + fadeOut()
        ) {
            Card(Modifier.fillMaxWidth()) {
                Text("上から展開", Modifier.padding(16.dp))
            }
        }
    }
}
```

---

## LazyColumnでのアニメーション

```kotlin
@Composable
fun AnimatedList() {
    var items by remember { mutableStateOf(List(5) { "Item $it" }) }

    Column(Modifier.padding(16.dp)) {
        Button(onClick = { items = items + "Item ${items.size}" }) {
            Text("追加")
        }

        LazyColumn {
            items(items, key = { it }) { item ->
                AnimatedVisibility(
                    visible = true,
                    enter = fadeIn() + expandVertically(),
                    modifier = Modifier.animateItem()
                ) {
                    ListItem(
                        headlineContent = { Text(item) },
                        trailingContent = {
                            IconButton(onClick = { items = items - item }) {
                                Icon(Icons.Default.Delete, "削除")
                            }
                        }
                    )
                }
            }
        }
    }
}
```

---

## まとめ

| Transition | 用途 |
|-----------|------|
| `fadeIn/fadeOut` | フェード |
| `slideIn/slideOut` | スライド |
| `expandVertically` | 展開 |
| `scaleIn/scaleOut` | スケール |

- `AnimatedVisibility`で表示/非表示をアニメーション
- `enter`/`exit`で入退場アニメーション指定
- `+`でアニメーションを組み合わせ
- `animateItem()`でLazyColumnのアイテムアニメーション

---

8種類のAndroidアプリテンプレート（アニメーション対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [アニメーション](https://zenn.dev/myougatheaxo/articles/android-compose-animation-spring-2026)
- [AnimatedContent](https://zenn.dev/myougatheaxo/articles/android-compose-compose-animated-content-2026)
- [SharedElementTransition](https://zenn.dev/myougatheaxo/articles/android-compose-shared-element-transition-2026)
