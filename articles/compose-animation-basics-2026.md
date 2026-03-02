---
title: "Compose Animation入門 — 3行でUIに動きをつける4つの方法"
emoji: "✨"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "animation"]
published: true
---

## この記事で学べること

Jetpack Composeでは、**たった3行でアニメーションが追加**できます。XMLの頃のアニメーション地獄はもう過去の話。

4つの基本アニメーションを紹介します。

---

## 1. animateFloatAsState — 値の変化をアニメーション

```kotlin
@Composable
fun ProgressBar(progress: Float) {
    val animatedProgress by animateFloatAsState(
        targetValue = progress,
        animationSpec = tween(durationMillis = 500)
    )

    LinearProgressIndicator(progress = { animatedProgress })
}
```

`progress`の値が変わると、自動的にアニメーションで遷移します。

他の型にも対応：
- `animateColorAsState` — 色の変化
- `animateDpAsState` — サイズの変化
- `animateIntAsState` — 整数の変化

---

## 2. AnimatedVisibility — 表示/非表示をアニメーション

```kotlin
@Composable
fun ToggleMessage(visible: Boolean) {
    AnimatedVisibility(
        visible = visible,
        enter = fadeIn() + slideInVertically(),
        exit = fadeOut() + slideOutVertically()
    ) {
        Text("こんにちは！", style = MaterialTheme.typography.headlineMedium)
    }
}
```

`visible`がtrueになるとフェードイン+スライドで表示。falseでフェードアウト。

### 入場・退場アニメーションの組み合わせ

| 入場 | 退場 | 効果 |
|------|------|------|
| `fadeIn()` | `fadeOut()` | フェード |
| `slideInVertically()` | `slideOutVertically()` | 縦スライド |
| `slideInHorizontally()` | `slideOutHorizontally()` | 横スライド |
| `expandVertically()` | `shrinkVertically()` | 展開/収縮 |
| `scaleIn()` | `scaleOut()` | 拡大/縮小 |

`+`で組み合わせ可能。`fadeIn() + slideInVertically()`でフェード+スライドが同時に動く。

---

## 3. animateContentSize — サイズ変更をアニメーション

```kotlin
@Composable
fun ExpandableCard() {
    var expanded by remember { mutableStateOf(false) }

    Card(
        modifier = Modifier
            .animateContentSize()
            .clickable { expanded = !expanded }
    ) {
        Column(Modifier.padding(16.dp)) {
            Text("タイトル", style = MaterialTheme.typography.titleMedium)
            if (expanded) {
                Text("詳細テキストがここに表示されます。タップで開閉。")
            }
        }
    }
}
```

`Modifier.animateContentSize()`を付けるだけ。コンテンツのサイズが変わると自動でアニメーション。

---

## 4. Crossfade — コンテンツ切り替えをアニメーション

```kotlin
@Composable
fun TabContent(selectedTab: Int) {
    Crossfade(targetState = selectedTab, label = "tab") { tab ->
        when (tab) {
            0 -> HomeScreen()
            1 -> SettingsScreen()
            2 -> ProfileScreen()
        }
    }
}
```

タブ切り替え時にフェードでコンテンツが入れ替わります。

---

## 実践例：FABの回転アニメーション

```kotlin
@Composable
fun RotatingFab(onClick: () -> Unit) {
    var isOpen by remember { mutableStateOf(false) }
    val rotation by animateFloatAsState(
        targetValue = if (isOpen) 45f else 0f,
        animationSpec = tween(300)
    )

    FloatingActionButton(
        onClick = {
            isOpen = !isOpen
            onClick()
        }
    ) {
        Icon(
            Icons.Default.Add,
            contentDescription = "追加",
            modifier = Modifier.rotate(rotation)
        )
    }
}
```

FABをタップすると+アイコンが45度回転して×に見える定番パターン。

---

## アニメーションの選び方

| やりたいこと | 使うAPI |
|------------|---------|
| 値を滑らかに変化させたい | `animate*AsState` |
| 表示/非表示を切り替えたい | `AnimatedVisibility` |
| サイズ変更を滑らかにしたい | `animateContentSize` |
| コンテンツを切り替えたい | `Crossfade` |
| 複雑なアニメーション | `updateTransition` |

---

## まとめ

- `animateFloatAsState` — 値の変化を3行でアニメーション
- `AnimatedVisibility` — 表示/非表示をフェード+スライド
- `animateContentSize` — Modifierを1つ追加するだけ
- `Crossfade` — コンテンツ切り替えをフェード

---

8種類のAndroidアプリテンプレート（Compose UI + カスタマイズ自由）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Material3テーマ完全ガイド](https://zenn.dev/myougatheaxo/articles/material3-theme-guide-2026)
- [ViewModel + StateFlow入門](https://zenn.dev/myougatheaxo/articles/android-viewmodel-stateflow-2026)
- [Compose Navigation入門](https://zenn.dev/myougatheaxo/articles/compose-navigation-guide-2026)
- [AIでAndroidアプリを47秒で作った話](https://zenn.dev/myougatheaxo/articles/ai-android-app-47seconds)
