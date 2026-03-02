---
title: "Compose for TV完全ガイド — TvLazyRow/フォーカス管理/リモコン操作"
emoji: "📺"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "androidtv"]
published: true
---

## この記事で学べること

**Compose for TV**（TvLazyRow、フォーカス管理、リモコン操作、D-Pad対応レイアウト）を解説します。

---

## セットアップ

```groovy
// build.gradle
dependencies {
    implementation("androidx.tv:tv-foundation:1.0.0-alpha11")
    implementation("androidx.tv:tv-material:1.0.0-rc01")
}
```

---

## TV向けレイアウト

```kotlin
@Composable
fun TvHomeScreen() {
    val categories = listOf("おすすめ", "新着", "人気", "アクション")

    TvLazyColumn(
        modifier = Modifier.fillMaxSize(),
        contentPadding = PaddingValues(vertical = 24.dp)
    ) {
        items(categories) { category ->
            Text(category, Modifier.padding(start = 48.dp, bottom = 8.dp),
                style = MaterialTheme.typography.titleLarge)

            TvLazyRow(
                contentPadding = PaddingValues(horizontal = 48.dp),
                horizontalArrangement = Arrangement.spacedBy(16.dp)
            ) {
                items(10) { index ->
                    ContentCard(title = "$category #${index + 1}")
                }
            }

            Spacer(Modifier.height(24.dp))
        }
    }
}

@Composable
fun ContentCard(title: String) {
    var isFocused by remember { mutableStateOf(false) }

    Card(
        onClick = { /* コンテンツ再生 */ },
        modifier = Modifier
            .size(width = 200.dp, height = 120.dp)
            .onFocusChanged { isFocused = it.isFocused }
            .then(if (isFocused) Modifier.border(3.dp, MaterialTheme.colorScheme.primary,
                RoundedCornerShape(8.dp)) else Modifier)
    ) {
        Box(Modifier.fillMaxSize(), contentAlignment = Alignment.BottomStart) {
            Text(title, Modifier.padding(12.dp), style = MaterialTheme.typography.bodyMedium)
        }
    }
}
```

---

## リモコンキー操作

```kotlin
@Composable
fun DpadNavigationDemo() {
    var selectedIndex by remember { mutableIntStateOf(0) }
    val items = listOf("再生", "一時停止", "停止", "次へ", "前へ")

    Row(
        Modifier.fillMaxWidth()
            .onKeyEvent { event ->
                when (event.key) {
                    Key.DirectionRight -> {
                        selectedIndex = (selectedIndex + 1).coerceAtMost(items.lastIndex)
                        true
                    }
                    Key.DirectionLeft -> {
                        selectedIndex = (selectedIndex - 1).coerceAtLeast(0)
                        true
                    }
                    Key.Enter, Key.DirectionCenter -> {
                        // 選択実行
                        true
                    }
                    else -> false
                }
            },
        horizontalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        items.forEachIndexed { index, label ->
            FilterChip(
                selected = index == selectedIndex,
                onClick = { selectedIndex = index },
                label = { Text(label) }
            )
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `TvLazyRow` | TV向け横スクロール |
| `TvLazyColumn` | TV向け縦スクロール |
| `onFocusChanged` | フォーカス検出 |
| `onKeyEvent` | リモコンキー入力 |

- Compose for TVはD-Padフォーカス管理が重要
- `TvLazyRow`/`TvLazyColumn`でTV最適化レイアウト
- フォーカス状態でカードのハイライト表示
- `onKeyEvent`でリモコンの方向キー操作を処理

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose WearOS](https://zenn.dev/myougatheaxo/articles/android-compose-compose-wear-os-2026)
- [Compose Foldable](https://zenn.dev/myougatheaxo/articles/android-compose-compose-foldable-2026)
- [Compose WindowSizeClass](https://zenn.dev/myougatheaxo/articles/android-compose-compose-window-size-class-2026)
