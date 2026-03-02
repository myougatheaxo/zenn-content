---
title: "Material3 Motionガイド — コンテナ変形/共有軸/フェードスルー"
emoji: "✨"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "animation"]
published: true
---

## この記事で学べること

**Material3 Motion**（コンテナ変形、共有軸遷移、フェードスルー、クロスフェード、Composeアニメーション実装）を解説します。

---

## Material Motionパターン

| パターン | 用途 |
|---------|------|
| Container Transform | カード→詳細画面 |
| Shared Axis | 同階層のステップ遷移 |
| Fade Through | 無関係な画面切替 |
| Fade | 表示/非表示 |

---

## Container Transform (共有要素)

```kotlin
@Composable
fun ContainerTransformExample() {
    SharedTransitionLayout {
        AnimatedContent(
            targetState = showDetail,
            transitionSpec = {
                fadeIn(tween(300)) togetherWith fadeOut(tween(300))
            }
        ) { isDetail ->
            if (!isDetail) {
                Card(
                    Modifier
                        .sharedBounds(
                            rememberSharedContentState(key = "card"),
                            animatedVisibilityScope = this@AnimatedContent,
                            resizeMode = SharedTransitionScope.ResizeMode.RemeasureToBounds
                        )
                        .size(100.dp)
                        .clickable { showDetail = true }
                ) { Text("Card") }
            } else {
                Surface(
                    Modifier
                        .sharedBounds(
                            rememberSharedContentState(key = "card"),
                            animatedVisibilityScope = this@AnimatedContent
                        )
                        .fillMaxSize()
                ) {
                    Column(Modifier.padding(16.dp)) {
                        Text("詳細画面", style = MaterialTheme.typography.headlineMedium)
                        Button(onClick = { showDetail = false }) { Text("戻る") }
                    }
                }
            }
        }
    }
}
```

---

## Shared Axis（共有軸遷移）

```kotlin
@Composable
fun SharedAxisTransition(currentStep: Int) {
    AnimatedContent(
        targetState = currentStep,
        transitionSpec = {
            if (targetState > initialState) {
                (slideInHorizontally { it } + fadeIn()) togetherWith
                    (slideOutHorizontally { -it } + fadeOut())
            } else {
                (slideInHorizontally { -it } + fadeIn()) togetherWith
                    (slideOutHorizontally { it } + fadeOut())
            }.using(SizeTransform(clip = false))
        }
    ) { step ->
        when (step) {
            0 -> StepContent("ステップ1: 基本情報")
            1 -> StepContent("ステップ2: 詳細設定")
            2 -> StepContent("ステップ3: 確認")
        }
    }
}
```

---

## Fade Through（フェードスルー）

```kotlin
@Composable
fun FadeThroughTransition(selectedTab: Int) {
    AnimatedContent(
        targetState = selectedTab,
        transitionSpec = {
            fadeIn(tween(220, delayMillis = 90)) togetherWith
                fadeOut(tween(90))
        }
    ) { tab ->
        when (tab) {
            0 -> HomeContent()
            1 -> SearchContent()
            2 -> ProfileContent()
        }
    }
}
```

---

## AnimatedVisibility（表示/非表示）

```kotlin
@Composable
fun FabWithAnimation(isVisible: Boolean, onClick: () -> Unit) {
    AnimatedVisibility(
        visible = isVisible,
        enter = scaleIn(
            initialScale = 0.6f,
            animationSpec = tween(220, easing = FastOutSlowInEasing)
        ) + fadeIn(tween(220)),
        exit = scaleOut(
            targetScale = 0.6f,
            animationSpec = tween(220)
        ) + fadeOut(tween(220))
    ) {
        FloatingActionButton(onClick = onClick) {
            Icon(Icons.Default.Add, "追加")
        }
    }
}
```

---

## まとめ

| パターン | Compose API |
|---------|-------------|
| Container Transform | `SharedTransitionLayout` + `sharedBounds` |
| Shared Axis | `AnimatedContent` + `slideIn/Out` |
| Fade Through | `AnimatedContent` + `fadeIn/Out` |
| Fade | `AnimatedVisibility` |

- Material Motionガイドラインに沿ったアニメーション
- `SharedTransitionLayout`でコンテナ変形
- `AnimatedContent`でコンテンツ切替アニメーション
- `tween`/`spring`でイージング調整

---

8種類のAndroidアプリテンプレート（Material3 Motion対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [アニメーション基本](https://zenn.dev/myougatheaxo/articles/android-compose-animation-transition-2026)
- [SharedElement](https://zenn.dev/myougatheaxo/articles/android-compose-shared-element-2026)
- [Navigation遷移](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-animation-2026)
