---
title: "MotionLayout完全ガイド — アニメーション遷移/ConstraintSet/トランジション"
emoji: "🎬"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "animation"]
published: true
---

## この記事で学べること

**MotionLayout**（ConstraintSet間のアニメーション遷移、スクロール連動、コラプシングヘッダー）を解説します。

---

## MotionLayout基本

```kotlin
@OptIn(ExperimentalMotionApi::class)
@Composable
fun MotionLayoutBasic() {
    var progress by remember { mutableFloatStateOf(0f) }

    val startSet = ConstraintSet("""{
        header: { top: 'parent', start: 'parent', end: 'parent', height: 200 },
        title: { bottom: 'header', start: 'header', end: 'header' },
        avatar: { top: 'header', start: 'parent', width: 80, height: 80 }
    }""")

    val endSet = ConstraintSet("""{
        header: { top: 'parent', start: 'parent', end: 'parent', height: 56 },
        title: { top: 'header', bottom: 'header', start: 'avatar', end: 'parent' },
        avatar: { top: 'header', bottom: 'header', start: 'parent', width: 40, height: 40 }
    }""")

    Column {
        MotionLayout(
            start = startSet, end = endSet,
            progress = progress,
            modifier = Modifier.fillMaxWidth()
        ) {
            Box(Modifier.layoutId("header").background(MaterialTheme.colorScheme.primary))
            Text("プロフィール", Modifier.layoutId("title"), color = Color.White, fontWeight = FontWeight.Bold)
            Box(Modifier.layoutId("avatar").clip(CircleShape).background(Color.White))
        }

        Slider(value = progress, onValueChange = { progress = it }, Modifier.padding(16.dp))
    }
}
```

---

## コラプシングツールバー

```kotlin
@OptIn(ExperimentalMotionApi::class)
@Composable
fun CollapsingToolbar() {
    val scrollState = rememberScrollState()
    val progress = min(1f, scrollState.value / 500f)

    val startSet = ConstraintSet("""{
        toolbar: { top: 'parent', start: 'parent', end: 'parent', height: 250 },
        title: { bottom: 'toolbar', start: 'parent', fontSize: 24 },
        image: { top: 'parent', start: 'parent', end: 'parent', bottom: 'toolbar', alpha: 1.0 }
    }""")

    val endSet = ConstraintSet("""{
        toolbar: { top: 'parent', start: 'parent', end: 'parent', height: 64 },
        title: { top: 'toolbar', bottom: 'toolbar', start: 'parent', fontSize: 18 },
        image: { top: 'parent', start: 'parent', end: 'parent', height: 64, alpha: 0.0 }
    }""")

    Column(Modifier.fillMaxSize()) {
        MotionLayout(
            start = startSet, end = endSet,
            progress = progress,
            modifier = Modifier.fillMaxWidth()
        ) {
            Box(Modifier.layoutId("toolbar").background(MaterialTheme.colorScheme.primary))
            AsyncImage(
                model = "https://example.com/header.jpg", null,
                Modifier.layoutId("image"), contentScale = ContentScale.Crop
            )
            Text("マイページ", Modifier.layoutId("title").padding(horizontal = 16.dp), color = Color.White)
        }

        Column(Modifier.verticalScroll(scrollState).padding(16.dp)) {
            repeat(20) { Text("コンテンツ ${it + 1}", Modifier.padding(8.dp)) }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `MotionLayout` | ConstraintSet間遷移 |
| `ConstraintSet` | レイアウト制約定義 |
| `progress` | 遷移進行度(0-1) |
| `layoutId` | 要素の識別 |

- `MotionLayout`で2つのConstraintSet間をアニメーション
- `progress`でスクロール連動の遷移制御
- コラプシングツールバーの実装に最適
- JSON形式でConstraintSetを定義

---

8種類のAndroidアプリテンプレート（アニメーション対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ConstraintLayout](https://zenn.dev/myougatheaxo/articles/android-compose-compose-constraint-layout-2026)
- [AnimatedVisibility](https://zenn.dev/myougatheaxo/articles/android-compose-compose-animated-visibility-2026)
- [Parallax](https://zenn.dev/myougatheaxo/articles/android-compose-compose-parallax-2026)
