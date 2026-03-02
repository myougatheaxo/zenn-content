---
title: "AnimatedContent完全ガイド — ContentTransform/Crossfade/ターゲット切り替え"
emoji: "🔀"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "animation"]
published: true
---

## この記事で学べること

**AnimatedContent**（ContentTransform、Crossfade、ターゲット状態切り替えアニメーション）を解説します。

---

## 基本AnimatedContent

```kotlin
@Composable
fun AnimatedContentExample() {
    var count by remember { mutableIntStateOf(0) }

    Column(Modifier.padding(16.dp), horizontalAlignment = Alignment.CenterHorizontally) {
        AnimatedContent(
            targetState = count,
            transitionSpec = {
                if (targetState > initialState) {
                    slideInVertically { -it } + fadeIn() togetherWith
                        slideOutVertically { it } + fadeOut()
                } else {
                    slideInVertically { it } + fadeIn() togetherWith
                        slideOutVertically { -it } + fadeOut()
                }.using(SizeTransform(clip = false))
            },
            label = "counter"
        ) { target ->
            Text("$target", fontSize = 64.sp, fontWeight = FontWeight.Bold)
        }

        Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            Button(onClick = { count-- }) { Text("-") }
            Button(onClick = { count++ }) { Text("+") }
        }
    }
}
```

---

## Crossfade

```kotlin
@Composable
fun CrossfadeExample() {
    var currentScreen by remember { mutableStateOf("home") }

    Column {
        Row(Modifier.padding(16.dp), horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            listOf("home", "search", "profile").forEach { screen ->
                FilterChip(
                    selected = currentScreen == screen,
                    onClick = { currentScreen = screen },
                    label = { Text(screen) }
                )
            }
        }

        Crossfade(targetState = currentScreen, label = "screen") { screen ->
            when (screen) {
                "home" -> Box(Modifier.fillMaxSize().background(Color(0xFFE3F2FD))) {
                    Text("ホーム", Modifier.padding(16.dp))
                }
                "search" -> Box(Modifier.fillMaxSize().background(Color(0xFFF3E5F5))) {
                    Text("検索", Modifier.padding(16.dp))
                }
                "profile" -> Box(Modifier.fillMaxSize().background(Color(0xFFE8F5E9))) {
                    Text("プロフィール", Modifier.padding(16.dp))
                }
            }
        }
    }
}
```

---

## 状態別UI切り替え

```kotlin
sealed class UiState {
    data object Loading : UiState()
    data class Success(val data: List<String>) : UiState()
    data class Error(val message: String) : UiState()
}

@Composable
fun StateAnimatedContent(state: UiState) {
    AnimatedContent(
        targetState = state,
        transitionSpec = {
            fadeIn(tween(300)) togetherWith fadeOut(tween(300))
        },
        label = "state"
    ) { targetState ->
        when (targetState) {
            is UiState.Loading -> {
                Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                    CircularProgressIndicator()
                }
            }
            is UiState.Success -> {
                LazyColumn {
                    items(targetState.data) { item ->
                        ListItem(headlineContent = { Text(item) })
                    }
                }
            }
            is UiState.Error -> {
                Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                    Text(targetState.message, color = MaterialTheme.colorScheme.error)
                }
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `AnimatedContent` | 状態切り替えアニメーション |
| `Crossfade` | クロスフェード |
| `ContentTransform` | 入退場の組み合わせ |
| `SizeTransform` | サイズ変化アニメーション |

- `AnimatedContent`で状態変化時のアニメーション
- `transitionSpec`で方向別アニメーション定義
- `Crossfade`でシンプルなフェード切り替え
- `SizeTransform`でサイズ変化もアニメーション

---

8種類のAndroidアプリテンプレート（アニメーション対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [AnimatedVisibility](https://zenn.dev/myougatheaxo/articles/android-compose-compose-animated-visibility-2026)
- [アニメーション](https://zenn.dev/myougatheaxo/articles/android-compose-animation-spring-2026)
- [SharedElementTransition](https://zenn.dev/myougatheaxo/articles/android-compose-shared-element-transition-2026)
