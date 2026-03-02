---
title: "ConstraintLayout in Compose — 複雑なレイアウトを宣言的に構築"
emoji: "📐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**ConstraintLayout**をComposeで使い、複雑なレイアウトをフラットに構築する方法を解説します。

---

## セットアップ

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.constraintlayout:constraintlayout-compose:1.1.0")
}
```

---

## 基本のConstraintLayout

```kotlin
@Composable
fun ProfileCard() {
    ConstraintLayout(
        modifier = Modifier.fillMaxWidth().padding(16.dp)
    ) {
        val (avatar, name, bio, button) = createRefs()

        Image(
            painter = painterResource(R.drawable.avatar),
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
            "みょうが",
            style = MaterialTheme.typography.titleLarge,
            modifier = Modifier.constrainAs(name) {
                top.linkTo(avatar.top)
                start.linkTo(avatar.end, margin = 16.dp)
            }
        )

        Text(
            "ウーパールーパーVTuber",
            style = MaterialTheme.typography.bodyMedium,
            modifier = Modifier.constrainAs(bio) {
                top.linkTo(name.bottom, margin = 4.dp)
                start.linkTo(name.start)
            }
        )

        Button(
            onClick = {},
            modifier = Modifier.constrainAs(button) {
                top.linkTo(avatar.top)
                bottom.linkTo(avatar.bottom)
                end.linkTo(parent.end)
            }
        ) {
            Text("フォロー")
        }
    }
}
```

---

## ガイドライン

```kotlin
ConstraintLayout(Modifier.fillMaxSize()) {
    val (title, content) = createRefs()
    val guideline = createGuidelineFromTop(0.3f)

    Text(
        "ヘッダー",
        modifier = Modifier.constrainAs(title) {
            bottom.linkTo(guideline)
            centerHorizontallyTo(parent)
        }
    )

    Text(
        "コンテンツ",
        modifier = Modifier.constrainAs(content) {
            top.linkTo(guideline, margin = 16.dp)
            centerHorizontallyTo(parent)
        }
    )
}
```

`createGuidelineFromTop(fraction)`でレイアウト位置の基準線を作成。

---

## バリア

```kotlin
ConstraintLayout(Modifier.fillMaxWidth().padding(16.dp)) {
    val (label1, label2, value) = createRefs()
    val barrier = createEndBarrier(label1, label2)

    Text("名前", modifier = Modifier.constrainAs(label1) {
        top.linkTo(parent.top)
        start.linkTo(parent.start)
    })

    Text("メールアドレス", modifier = Modifier.constrainAs(label2) {
        top.linkTo(label1.bottom, margin = 8.dp)
        start.linkTo(parent.start)
    })

    Text("value...", modifier = Modifier.constrainAs(value) {
        top.linkTo(parent.top)
        start.linkTo(barrier, margin = 16.dp)
    })
}
```

バリアは複数要素の端を基準にして配置。

---

## チェーン

```kotlin
ConstraintLayout(Modifier.fillMaxWidth().padding(16.dp)) {
    val (btn1, btn2, btn3) = createRefs()
    createHorizontalChain(btn1, btn2, btn3, chainStyle = ChainStyle.Spread)

    Button(onClick = {}, modifier = Modifier.constrainAs(btn1) {
        top.linkTo(parent.top)
    }) { Text("A") }

    Button(onClick = {}, modifier = Modifier.constrainAs(btn2) {
        top.linkTo(parent.top)
    }) { Text("B") }

    Button(onClick = {}, modifier = Modifier.constrainAs(btn3) {
        top.linkTo(parent.top)
    }) { Text("C") }
}
```

チェーンスタイル:
- `ChainStyle.Spread` — 均等配置
- `ChainStyle.SpreadInside` — 両端に寄せて均等
- `ChainStyle.Packed` — 中央に詰めて配置

---

## 幅の制約

```kotlin
Text(
    "長いテキストを制約付きで表示...",
    modifier = Modifier.constrainAs(text) {
        start.linkTo(avatar.end, margin = 16.dp)
        end.linkTo(parent.end, margin = 16.dp)
        width = Dimension.fillToConstraints  // 制約に合わせて幅を決定
    },
    maxLines = 2,
    overflow = TextOverflow.Ellipsis
)
```

- `Dimension.fillToConstraints` — 制約に合わせる
- `Dimension.wrapContent` — コンテンツに合わせる
- `Dimension.preferredWrapContent` — 制約内でwrap

---

## ConstraintSet（分離定義）

```kotlin
@Composable
fun DecoupledLayout() {
    val constraints = ConstraintSet {
        val title = createRefFor("title")
        val subtitle = createRefFor("subtitle")

        constrain(title) {
            top.linkTo(parent.top, margin = 16.dp)
            centerHorizontallyTo(parent)
        }
        constrain(subtitle) {
            top.linkTo(title.bottom, margin = 8.dp)
            centerHorizontallyTo(parent)
        }
    }

    ConstraintLayout(constraints, Modifier.fillMaxSize()) {
        Text("タイトル", Modifier.layoutId("title"),
            style = MaterialTheme.typography.headlineMedium)
        Text("サブタイトル", Modifier.layoutId("subtitle"),
            style = MaterialTheme.typography.bodyLarge)
    }
}
```

---

## まとめ

- `ConstraintLayout`でフラットに複雑レイアウト
- `constrainAs`で制約を宣言
- ガイドライン・バリア・チェーンで位置関係を定義
- `Dimension.fillToConstraints`で幅制御
- `ConstraintSet`で制約とUIを分離

---

8種類のAndroidアプリテンプレート（最適なレイアウト設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Composeレイアウト入門](https://zenn.dev/myougatheaxo/articles/compose-layout-basics-2026)
- [Adaptive Layout完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-adaptive-layout-2026)
- [LazyVerticalGrid完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-lazy-grid-2026)
