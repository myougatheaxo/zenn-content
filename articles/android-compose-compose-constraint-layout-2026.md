---
title: "ConstraintLayout完全ガイド — 制約/チェーン/バリア/ガイドライン"
emoji: "📐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "layout"]
published: true
---

## この記事で学べること

**ConstraintLayout**（制約定義、チェーン、バリア、ガイドライン）を解説します。

---

## 基本制約

```kotlin
@Composable
fun ConstraintLayoutBasic() {
    ConstraintLayout(Modifier.fillMaxWidth().padding(16.dp)) {
        val (avatar, name, email, button) = createRefs()

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
            "山田太郎",
            style = MaterialTheme.typography.titleMedium,
            modifier = Modifier.constrainAs(name) {
                top.linkTo(avatar.top)
                start.linkTo(avatar.end, margin = 12.dp)
            }
        )

        Text(
            "taro@example.com",
            style = MaterialTheme.typography.bodySmall,
            modifier = Modifier.constrainAs(email) {
                top.linkTo(name.bottom, margin = 4.dp)
                start.linkTo(name.start)
            }
        )

        Button(
            onClick = {},
            modifier = Modifier.constrainAs(button) {
                top.linkTo(parent.top)
                bottom.linkTo(parent.bottom)
                end.linkTo(parent.end)
            }
        ) { Text("編集") }
    }
}
```

---

## ガイドライン・バリア

```kotlin
@Composable
fun GuidelineBarrierExample() {
    ConstraintLayout(Modifier.fillMaxSize().padding(16.dp)) {
        val (label1, value1, label2, value2) = createRefs()
        val guideline = createGuidelineFromStart(0.3f)
        val barrier = createEndBarrier(label1, label2)

        Text("名前:", Modifier.constrainAs(label1) {
            top.linkTo(parent.top)
            end.linkTo(guideline)
        })
        Text("山田太郎", Modifier.constrainAs(value1) {
            top.linkTo(label1.top)
            start.linkTo(barrier, margin = 8.dp)
        })
        Text("メール:", Modifier.constrainAs(label2) {
            top.linkTo(label1.bottom, margin = 12.dp)
            end.linkTo(guideline)
        })
        Text("taro@example.com", Modifier.constrainAs(value2) {
            top.linkTo(label2.top)
            start.linkTo(barrier, margin = 8.dp)
        })
    }
}
```

---

## チェーン

```kotlin
@Composable
fun ChainExample() {
    ConstraintLayout(Modifier.fillMaxWidth().padding(16.dp)) {
        val (btn1, btn2, btn3) = createRefs()

        createHorizontalChain(btn1, btn2, btn3, chainStyle = ChainStyle.SpreadInside)

        Button(onClick = {}, Modifier.constrainAs(btn1) {
            top.linkTo(parent.top)
        }) { Text("A") }
        Button(onClick = {}, Modifier.constrainAs(btn2) {
            top.linkTo(parent.top)
        }) { Text("B") }
        Button(onClick = {}, Modifier.constrainAs(btn3) {
            top.linkTo(parent.top)
        }) { Text("C") }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `constrainAs` | 制約定義 |
| `createGuidelineFrom*` | ガイドライン |
| `create*Barrier` | バリア |
| `createHorizontalChain` | チェーン |

- `ConstraintLayout`で複雑なレイアウトを宣言的に
- ガイドラインで割合ベースの位置決め
- バリアで複数要素の端に揃える
- チェーンで要素の均等配置

---

8種類のAndroidアプリテンプレート（レイアウト対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [MotionLayout](https://zenn.dev/myougatheaxo/articles/android-compose-compose-motion-layout-2026)
- [Adaptive Layout](https://zenn.dev/myougatheaxo/articles/android-compose-compose-adaptive-layout-2026)
- [LazyLayout](https://zenn.dev/myougatheaxo/articles/android-compose-compose-lazy-layout-2026)
