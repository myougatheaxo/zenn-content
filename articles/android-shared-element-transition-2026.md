---
title: "Compose SharedElement遷移ガイド — 画面間でアニメーション接続"
emoji: "🔀"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "animation"]
published: true
---

## この記事で学べること

Compose の **SharedTransitionLayout** を使って、画面間でシームレスなアニメーション遷移を実装する方法を解説します。

---

## セットアップ

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.compose.animation:animation:1.7.0")
    implementation("androidx.navigation:navigation-compose:2.8.0")
}
```

---

## 基本のShared Element遷移

```kotlin
@Composable
fun AppNavigation() {
    SharedTransitionLayout {
        val navController = rememberNavController()

        NavHost(navController, startDestination = "list") {
            composable("list") {
                ListScreen(
                    onItemClick = { item ->
                        navController.navigate("detail/${item.id}")
                    },
                    animatedVisibilityScope = this
                )
            }
            composable("detail/{id}") { backStackEntry ->
                val id = backStackEntry.arguments?.getString("id") ?: return@composable
                DetailScreen(
                    itemId = id,
                    onBack = { navController.popBackStack() },
                    animatedVisibilityScope = this
                )
            }
        }
    }
}
```

---

## リスト画面

```kotlin
@Composable
fun SharedTransitionScope.ListScreen(
    onItemClick: (Item) -> Unit,
    animatedVisibilityScope: AnimatedVisibilityScope
) {
    LazyColumn {
        items(sampleItems) { item ->
            Row(
                modifier = Modifier
                    .fillMaxWidth()
                    .clickable { onItemClick(item) }
                    .padding(16.dp),
                verticalAlignment = Alignment.CenterVertically
            ) {
                AsyncImage(
                    model = item.imageUrl,
                    contentDescription = null,
                    modifier = Modifier
                        .size(64.dp)
                        .clip(CircleShape)
                        .sharedElement(
                            state = rememberSharedContentState(key = "image-${item.id}"),
                            animatedVisibilityScope = animatedVisibilityScope
                        ),
                    contentScale = ContentScale.Crop
                )

                Spacer(Modifier.width(16.dp))

                Text(
                    item.title,
                    style = MaterialTheme.typography.titleMedium,
                    modifier = Modifier.sharedElement(
                        state = rememberSharedContentState(key = "title-${item.id}"),
                        animatedVisibilityScope = animatedVisibilityScope
                    )
                )
            }
        }
    }
}
```

---

## 詳細画面

```kotlin
@Composable
fun SharedTransitionScope.DetailScreen(
    itemId: String,
    onBack: () -> Unit,
    animatedVisibilityScope: AnimatedVisibilityScope
) {
    val item = sampleItems.first { it.id == itemId }

    Column(Modifier.fillMaxSize()) {
        AsyncImage(
            model = item.imageUrl,
            contentDescription = null,
            modifier = Modifier
                .fillMaxWidth()
                .height(300.dp)
                .sharedElement(
                    state = rememberSharedContentState(key = "image-${item.id}"),
                    animatedVisibilityScope = animatedVisibilityScope
                ),
            contentScale = ContentScale.Crop
        )

        Text(
            item.title,
            style = MaterialTheme.typography.headlineMedium,
            modifier = Modifier
                .padding(16.dp)
                .sharedElement(
                    state = rememberSharedContentState(key = "title-${item.id}"),
                    animatedVisibilityScope = animatedVisibilityScope
                )
        )

        Text(
            item.description,
            style = MaterialTheme.typography.bodyLarge,
            modifier = Modifier.padding(horizontal = 16.dp)
        )
    }
}
```

---

## sharedBounds（コンテナ遷移）

```kotlin
// カード全体をshared boundsに
Card(
    modifier = Modifier
        .sharedBounds(
            sharedContentState = rememberSharedContentState(key = "card-${item.id}"),
            animatedVisibilityScope = animatedVisibilityScope,
            resizeMode = SharedTransitionScope.ResizeMode.RemeasureToBounds
        )
        .clickable { onItemClick(item) }
) {
    // カード内容
}
```

`sharedElement`は要素単位、`sharedBounds`はコンテナ単位の遷移。

---

## アニメーションのカスタマイズ

```kotlin
Modifier.sharedElement(
    state = rememberSharedContentState(key = "image-${item.id}"),
    animatedVisibilityScope = animatedVisibilityScope,
    boundsTransform = { initial, target ->
        tween(durationMillis = 500, easing = FastOutSlowInEasing)
    }
)
```

---

## 遷移中の他要素のフェード

```kotlin
Text(
    item.description,
    modifier = Modifier
        .padding(16.dp)
        .renderInSharedTransitionScopeOverlay(
            zIndexInOverlay = 0f
        )
        .animateEnterExit(
            enter = fadeIn(tween(300, delayMillis = 200)),
            exit = fadeOut(tween(100))
        )
)
```

Shared Element遷移中に他の要素をフェードイン/アウト。

---

## まとめ

- `SharedTransitionLayout`で画面間アニメーション
- `sharedElement`で個別要素を接続
- `sharedBounds`でコンテナ単位の遷移
- `rememberSharedContentState(key)`でキーを一致させる
- `boundsTransform`でアニメーションカスタマイズ
- `animateEnterExit`で非共有要素のフェード制御

---

8種類のAndroidアプリテンプレート（画面遷移アニメーション対応設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Composeアニメーション入門](https://zenn.dev/myougatheaxo/articles/compose-animation-basics-2026)
- [Navigation完全ガイド](https://zenn.dev/myougatheaxo/articles/android-navigation-compose-2026)
- [Canvas完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-canvas-drawing-2026)
