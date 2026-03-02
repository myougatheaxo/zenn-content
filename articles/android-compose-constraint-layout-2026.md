---
title: "ConstraintLayout Compose完全ガイド — 制約/チェーン/バリア/ガイドライン"
emoji: "📐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "layout"]
published: true
---

## この記事で学べること

**ConstraintLayout Compose**（制約定義、チェーン、バリア、ガイドライン、複雑レイアウト）を解説します。

---

## 基本制約

```kotlin
@Composable
fun ProfileCard() {
    ConstraintLayout(
        Modifier.fillMaxWidth().padding(16.dp)
    ) {
        val (avatar, name, bio, followBtn) = createRefs()

        Image(
            painter = painterResource(R.drawable.avatar),
            contentDescription = null,
            modifier = Modifier
                .size(64.dp)
                .clip(CircleShape)
                .constrainAs(avatar) {
                    start.linkTo(parent.start)
                    top.linkTo(parent.top)
                }
        )

        Text(
            "ユーザー名",
            style = MaterialTheme.typography.titleMedium,
            modifier = Modifier.constrainAs(name) {
                start.linkTo(avatar.end, margin = 16.dp)
                top.linkTo(avatar.top)
            }
        )

        Text(
            "プロフィール説明文がここに入ります",
            modifier = Modifier.constrainAs(bio) {
                start.linkTo(name.start)
                top.linkTo(name.bottom, margin = 4.dp)
                end.linkTo(parent.end)
                width = Dimension.fillToConstraints
            }
        )

        Button(
            onClick = {},
            modifier = Modifier.constrainAs(followBtn) {
                end.linkTo(parent.end)
                top.linkTo(parent.top)
            }
        ) { Text("フォロー") }
    }
}
```

---

## チェーン/バリア

```kotlin
@Composable
fun ChainExample() {
    ConstraintLayout(Modifier.fillMaxWidth().padding(16.dp)) {
        val (btn1, btn2, btn3) = createRefs()

        createHorizontalChain(btn1, btn2, btn3, chainStyle = ChainStyle.SpreadInside)

        Button(onClick = {}, Modifier.constrainAs(btn1) {}) { Text("A") }
        Button(onClick = {}, Modifier.constrainAs(btn2) {}) { Text("B") }
        Button(onClick = {}, Modifier.constrainAs(btn3) {}) { Text("C") }
    }
}

@Composable
fun BarrierExample() {
    ConstraintLayout(Modifier.fillMaxWidth().padding(16.dp)) {
        val (label1, value1, label2, value2, divider) = createRefs()

        val barrier = createEndBarrier(label1, label2, margin = 16.dp)

        Text("名前", Modifier.constrainAs(label1) { start.linkTo(parent.start); top.linkTo(parent.top) })
        Text("太郎", Modifier.constrainAs(value1) { start.linkTo(barrier); top.linkTo(label1.top) })

        Text("メール", Modifier.constrainAs(label2) { start.linkTo(parent.start); top.linkTo(label1.bottom, 8.dp) })
        Text("taro@example.com", Modifier.constrainAs(value2) { start.linkTo(barrier); top.linkTo(label2.top) })
    }
}
```

---

## ガイドライン

```kotlin
@Composable
fun GuidelineExample() {
    ConstraintLayout(Modifier.fillMaxSize()) {
        val guideline = createGuidelineFromStart(0.3f)
        val (sidebar, content) = createRefs()

        Surface(
            color = MaterialTheme.colorScheme.surfaceVariant,
            modifier = Modifier.constrainAs(sidebar) {
                start.linkTo(parent.start)
                end.linkTo(guideline)
                top.linkTo(parent.top)
                bottom.linkTo(parent.bottom)
                width = Dimension.fillToConstraints
                height = Dimension.fillToConstraints
            }
        ) { Text("サイドバー", Modifier.padding(8.dp)) }

        Box(Modifier.constrainAs(content) {
            start.linkTo(guideline)
            end.linkTo(parent.end)
            top.linkTo(parent.top)
            bottom.linkTo(parent.bottom)
            width = Dimension.fillToConstraints
            height = Dimension.fillToConstraints
        }) { Text("コンテンツ", Modifier.padding(16.dp)) }
    }
}
```

---

## まとめ

| 機能 | API |
|------|-----|
| 制約 | `constrainAs` + `linkTo` |
| チェーン | `createHorizontalChain` |
| バリア | `createEndBarrier` |
| ガイドライン | `createGuidelineFromStart` |

- `ConstraintLayout`で複雑な相対配置
- チェーンで均等/詰め配置を実現
- バリアで動的なアライメント基準を作成
- ガイドラインで比率ベースのレイアウト分割

---

8種類のAndroidアプリテンプレート（レイアウト最適化済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [LazyLayout](https://zenn.dev/myougatheaxo/articles/android-compose-lazy-layout-2026)
- [Motion Layout](https://zenn.dev/myougatheaxo/articles/android-compose-motion-layout-2026)
- [横画面/タブレット](https://zenn.dev/myougatheaxo/articles/android-compose-landscape-tablet-2026)
