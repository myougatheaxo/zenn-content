---
title: "ConstraintLayout for Compose — 複雑レイアウト構築ガイド"
emoji: "📐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "layout"]
published: true
---

## この記事で学べること

**ConstraintLayout for Compose**（制約定義、Chain、Barrier、Guideline、MotionLayout）を解説します。

---

## セットアップ

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.constraintlayout:constraintlayout-compose:1.1.0")
}
```

---

## 基本の制約

```kotlin
@Composable
fun ProfileCard() {
    ConstraintLayout(
        modifier = Modifier.fillMaxWidth().padding(16.dp)
    ) {
        val (avatar, name, bio, followBtn) = createRefs()

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
            text = "ユーザー名",
            style = MaterialTheme.typography.titleMedium,
            modifier = Modifier.constrainAs(name) {
                top.linkTo(avatar.top)
                start.linkTo(avatar.end, margin = 12.dp)
            }
        )

        Text(
            text = "プロフィール説明文がここに入ります",
            style = MaterialTheme.typography.bodySmall,
            modifier = Modifier.constrainAs(bio) {
                top.linkTo(name.bottom, margin = 4.dp)
                start.linkTo(name.start)
                end.linkTo(parent.end)
                width = Dimension.fillToConstraints
            }
        )

        Button(
            onClick = {},
            modifier = Modifier.constrainAs(followBtn) {
                top.linkTo(avatar.top)
                end.linkTo(parent.end)
            }
        ) {
            Text("フォロー")
        }
    }
}
```

---

## Chain（等間隔配置）

```kotlin
@Composable
fun ChainExample() {
    ConstraintLayout(Modifier.fillMaxWidth().padding(16.dp)) {
        val (btn1, btn2, btn3) = createRefs()

        // 水平チェーン
        createHorizontalChain(btn1, btn2, btn3, chainStyle = ChainStyle.SpreadInside)

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
}

// ChainStyle:
// Spread: 等間隔（デフォルト）
// SpreadInside: 両端固定、内側等間隔
// Packed: 中央寄せ
```

---

## Barrier

```kotlin
@Composable
fun BarrierExample() {
    ConstraintLayout(Modifier.fillMaxWidth().padding(16.dp)) {
        val (label1, label2, value1, value2) = createRefs()

        // ラベルの右端にバリアを作成
        val barrier = createEndBarrier(label1, label2, margin = 8.dp)

        Text("名前", Modifier.constrainAs(label1) {
            top.linkTo(parent.top)
            start.linkTo(parent.start)
        })

        Text("メールアドレス", Modifier.constrainAs(label2) {
            top.linkTo(label1.bottom, margin = 8.dp)
            start.linkTo(parent.start)
        })

        Text("Alice", Modifier.constrainAs(value1) {
            top.linkTo(label1.top)
            start.linkTo(barrier) // バリアの右側に配置
        })

        Text("alice@example.com", Modifier.constrainAs(value2) {
            top.linkTo(label2.top)
            start.linkTo(barrier)
        })
    }
}
```

---

## Guideline

```kotlin
@Composable
fun GuidelineExample() {
    ConstraintLayout(Modifier.fillMaxSize()) {
        val (image, text) = createRefs()

        // 30%の位置にガイドライン
        val guideline = createGuidelineFromStart(fraction = 0.3f)

        Image(
            painter = painterResource(R.drawable.photo),
            contentDescription = null,
            modifier = Modifier.constrainAs(image) {
                top.linkTo(parent.top)
                start.linkTo(parent.start)
                end.linkTo(guideline)
                width = Dimension.fillToConstraints
                height = Dimension.ratio("1:1")
            }
        )

        Text(
            "コンテンツ説明",
            modifier = Modifier.constrainAs(text) {
                top.linkTo(parent.top)
                start.linkTo(guideline, margin = 16.dp)
                end.linkTo(parent.end)
                width = Dimension.fillToConstraints
            }
        )
    }
}
```

---

## DecoupledConstraintLayout

```kotlin
// 制約セットを分離（条件分岐で使える）
@Composable
fun ResponsiveLayout(isWide: Boolean) {
    val constraints = if (isWide) wideConstraints() else narrowConstraints()

    ConstraintLayout(
        constraintSet = constraints,
        modifier = Modifier.fillMaxSize()
    ) {
        Image(
            painter = painterResource(R.drawable.photo),
            contentDescription = null,
            modifier = Modifier.layoutId("image")
        )
        Text("説明テキスト", modifier = Modifier.layoutId("text"))
    }
}

private fun wideConstraints() = ConstraintSet {
    val image = createRefFor("image")
    val text = createRefFor("text")

    constrain(image) {
        start.linkTo(parent.start)
        top.linkTo(parent.top)
        width = Dimension.percent(0.4f)
    }
    constrain(text) {
        start.linkTo(image.end, margin = 16.dp)
        top.linkTo(parent.top)
    }
}

private fun narrowConstraints() = ConstraintSet {
    val image = createRefFor("image")
    val text = createRefFor("text")

    constrain(image) {
        start.linkTo(parent.start)
        end.linkTo(parent.end)
        top.linkTo(parent.top)
        width = Dimension.fillToConstraints
    }
    constrain(text) {
        start.linkTo(parent.start)
        top.linkTo(image.bottom, margin = 16.dp)
    }
}
```

---

## まとめ

| 機能 | 用途 |
|------|------|
| `constrainAs` | 個別の制約定義 |
| `createHorizontalChain` | 水平等間隔配置 |
| `createEndBarrier` | 要素の最大端に境界線 |
| `createGuidelineFromStart` | パーセント位置の基準線 |
| `ConstraintSet` | 制約の外部定義 |
| `Dimension.fillToConstraints` | 制約間を埋める |

- 複雑なレイアウトではRow/Columnよりパフォーマンス良好
- `Dimension.ratio("16:9")`でアスペクト比指定
- `DecoupledConstraintLayout`でレスポンシブ対応

---

8種類のAndroidアプリテンプレート（レイアウト最適化済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [カスタムレイアウト](https://zenn.dev/myougatheaxo/articles/android-compose-custom-layout-2026)
- [マルチスクリーン](https://zenn.dev/myougatheaxo/articles/android-compose-multi-screen-size-2026)
- [Modifierチートシート](https://zenn.dev/myougatheaxo/articles/android-compose-modifier-cheatsheet-2026)
