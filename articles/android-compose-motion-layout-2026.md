---
title: "MotionLayout Composeガイド — 複雑な制約アニメーション"
emoji: "🎞️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "animation"]
published: true
---

## この記事で学べること

**ConstraintLayout + MotionLayout**でComposeの制約ベースアニメーションを解説します。

---

## セットアップ

```kotlin
dependencies {
    implementation("androidx.constraintlayout:constraintlayout-compose:1.1.0")
}
```

---

## MotionLayout基本

```kotlin
@Composable
fun CollapsibleHeader() {
    var expanded by remember { mutableStateOf(true) }
    val progress by animateFloatAsState(
        targetValue = if (expanded) 0f else 1f,
        animationSpec = tween(500),
        label = "progress"
    )

    MotionLayout(
        motionScene = MotionScene {
            val title = createRefFor("title")
            val image = createRefFor("image")

            defaultTransition(
                from = constraintSet {
                    constrain(image) {
                        width = Dimension.fillToConstraints
                        height = Dimension.value(200.dp)
                        top.linkTo(parent.top)
                        start.linkTo(parent.start)
                        end.linkTo(parent.end)
                    }
                    constrain(title) {
                        top.linkTo(image.bottom, 16.dp)
                        start.linkTo(parent.start, 16.dp)
                    }
                },
                to = constraintSet {
                    constrain(image) {
                        width = Dimension.value(56.dp)
                        height = Dimension.value(56.dp)
                        top.linkTo(parent.top, 8.dp)
                        start.linkTo(parent.start, 16.dp)
                    }
                    constrain(title) {
                        top.linkTo(parent.top, 24.dp)
                        start.linkTo(image.end, 16.dp)
                    }
                }
            )
        },
        progress = progress,
        modifier = Modifier
            .fillMaxWidth()
            .clickable { expanded = !expanded }
    ) {
        AsyncImage(
            model = "https://example.com/image.jpg",
            contentDescription = null,
            modifier = Modifier.layoutId("image"),
            contentScale = ContentScale.Crop
        )
        Text(
            "タイトル",
            modifier = Modifier.layoutId("title"),
            style = MaterialTheme.typography.titleLarge
        )
    }
}
```

---

## ConstraintLayout in Compose

```kotlin
@Composable
fun ProfileCard() {
    ConstraintLayout(
        Modifier
            .fillMaxWidth()
            .padding(16.dp)
    ) {
        val (avatar, name, bio, followButton) = createRefs()

        AsyncImage(
            model = "avatar_url",
            contentDescription = null,
            modifier = Modifier
                .size(64.dp)
                .clip(CircleShape)
                .constrainAs(avatar) {
                    top.linkTo(parent.top)
                    start.linkTo(parent.start)
                }
        )

        Text(
            "ユーザー名",
            style = MaterialTheme.typography.titleMedium,
            modifier = Modifier.constrainAs(name) {
                top.linkTo(avatar.top)
                start.linkTo(avatar.end, 16.dp)
            }
        )

        Text(
            "プロフィール説明文",
            style = MaterialTheme.typography.bodyMedium,
            modifier = Modifier.constrainAs(bio) {
                top.linkTo(name.bottom, 4.dp)
                start.linkTo(name.start)
                end.linkTo(parent.end)
                width = Dimension.fillToConstraints
            }
        )

        Button(
            onClick = { },
            modifier = Modifier.constrainAs(followButton) {
                top.linkTo(avatar.top)
                end.linkTo(parent.end)
            }
        ) { Text("フォロー") }
    }
}
```

---

## まとめ

- `ConstraintLayout`で複雑なレイアウト配置
- `MotionLayout`で制約間のアニメーション遷移
- `createRefFor`で参照を作成し`constrainAs`で制約設定
- `progress`パラメータでアニメーション制御
- `Dimension.fillToConstraints`で可変幅
- 折りたたみヘッダー等の複雑なUI向け

---

8種類のAndroidアプリテンプレート（レイアウト設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ConstraintLayoutガイド](https://zenn.dev/myougatheaxo/articles/compose-constraint-layout-2026)
- [アニメーション完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-animation-advanced-2026)
- [カスタムレイアウトガイド](https://zenn.dev/myougatheaxo/articles/android-compose-custom-layout-2026)
