---
title: "Shared Element Transition in Compose — 画面間アニメーション"
emoji: "🔀"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "animation"]
published: true
---

## この記事で学べること

Compose Navigation 2.8+の**Shared Element Transition**（共有要素遷移）を解説します。

---

## セットアップ

```kotlin
dependencies {
    implementation("androidx.navigation:navigation-compose:2.8.0")
    implementation("androidx.compose.animation:animation:1.7.0")
}
```

---

## 基本のSharedTransitionLayout

```kotlin
@Composable
fun AppNavigation() {
    SharedTransitionLayout {
        val navController = rememberNavController()

        NavHost(navController, startDestination = "list") {
            composable("list") {
                ListScreen(
                    onItemClick = { id -> navController.navigate("detail/$id") },
                    animatedVisibilityScope = this
                )
            }
            composable("detail/{id}") { backStackEntry ->
                val id = backStackEntry.arguments?.getString("id") ?: return@composable
                DetailScreen(
                    id = id,
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
    onItemClick: (String) -> Unit,
    animatedVisibilityScope: AnimatedVisibilityScope
) {
    LazyColumn {
        items(sampleItems) { item ->
            Row(
                modifier = Modifier
                    .fillMaxWidth()
                    .clickable { onItemClick(item.id) }
                    .padding(16.dp)
            ) {
                AsyncImage(
                    model = item.imageUrl,
                    contentDescription = null,
                    modifier = Modifier
                        .size(64.dp)
                        .clip(RoundedCornerShape(8.dp))
                        .sharedElement(
                            rememberSharedContentState(key = "image_${item.id}"),
                            animatedVisibilityScope = animatedVisibilityScope
                        ),
                    contentScale = ContentScale.Crop
                )

                Spacer(Modifier.width(16.dp))

                Text(
                    item.title,
                    modifier = Modifier.sharedElement(
                        rememberSharedContentState(key = "title_${item.id}"),
                        animatedVisibilityScope = animatedVisibilityScope
                    ),
                    style = MaterialTheme.typography.titleMedium
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
    id: String,
    onBack: () -> Unit,
    animatedVisibilityScope: AnimatedVisibilityScope
) {
    val item = sampleItems.find { it.id == id } ?: return

    Column(Modifier.fillMaxSize()) {
        AsyncImage(
            model = item.imageUrl,
            contentDescription = null,
            modifier = Modifier
                .fillMaxWidth()
                .height(300.dp)
                .sharedElement(
                    rememberSharedContentState(key = "image_${item.id}"),
                    animatedVisibilityScope = animatedVisibilityScope
                ),
            contentScale = ContentScale.Crop
        )

        Text(
            item.title,
            modifier = Modifier
                .padding(16.dp)
                .sharedElement(
                    rememberSharedContentState(key = "title_${item.id}"),
                    animatedVisibilityScope = animatedVisibilityScope
                ),
            style = MaterialTheme.typography.headlineMedium
        )

        Text(
            item.description,
            modifier = Modifier.padding(horizontal = 16.dp),
            style = MaterialTheme.typography.bodyLarge
        )
    }
}
```

---

## まとめ

- `SharedTransitionLayout`で共有遷移のスコープを定義
- `sharedElement`でアニメーション対象要素をマーク
- `rememberSharedContentState(key)`で同一要素を紐付け
- `AnimatedVisibilityScope`をNavHost composableから取得
- 画像+テキスト等、複数要素を同時に遷移可能
- Navigation 2.8+で公式サポート

---

8種類のAndroidアプリテンプレート（画面遷移アニメーション実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ナビゲーション遷移アニメーション](https://zenn.dev/myougatheaxo/articles/android-compose-transition-animation-2026)
- [型安全ナビゲーション](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-typesafe-2026)
- [MotionLayout](https://zenn.dev/myougatheaxo/articles/android-compose-motion-layout-2026)
