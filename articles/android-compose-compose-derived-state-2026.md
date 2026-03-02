---
title: "derivedStateOf完全ガイド — 派生状態/パフォーマンス最適化/不要リコンポジション防止"
emoji: "🧮"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "state"]
published: true
---

## この記事で学べること

**derivedStateOf**（派生状態、不要なリコンポジション防止、パフォーマンス最適化）を解説します。

---

## derivedStateOf基本

```kotlin
@Composable
fun FilteredListScreen() {
    val items = remember { (1..100).map { "Item $it" } }
    var query by remember { mutableStateOf("") }

    // queryが変わった時のみフィルタ再計算
    val filtered by remember {
        derivedStateOf {
            if (query.isEmpty()) items
            else items.filter { it.contains(query, ignoreCase = true) }
        }
    }

    Column(Modifier.padding(16.dp)) {
        OutlinedTextField(value = query, onValueChange = { query = it }, label = { Text("検索") })
        Text("${filtered.size}件", Modifier.padding(vertical = 8.dp))
        LazyColumn { items(filtered) { ListItem(headlineContent = { Text(it) }) } }
    }
}
```

---

## スクロール位置判定

```kotlin
@Composable
fun ScrollToTopButton() {
    val listState = rememberLazyListState()

    // スクロール位置に基づく表示判定
    val showButton by remember {
        derivedStateOf { listState.firstVisibleItemIndex > 5 }
    }

    Box(Modifier.fillMaxSize()) {
        LazyColumn(state = listState) {
            items(100) { ListItem(headlineContent = { Text("Item $it") }) }
        }

        AnimatedVisibility(
            visible = showButton,
            modifier = Modifier.align(Alignment.BottomEnd).padding(16.dp)
        ) {
            val scope = rememberCoroutineScope()
            FloatingActionButton(onClick = { scope.launch { listState.animateScrollToItem(0) } }) {
                Icon(Icons.Default.ArrowUpward, "トップへ")
            }
        }
    }
}
```

---

## 複数状態の結合

```kotlin
@Composable
fun FormValidation() {
    var name by remember { mutableStateOf("") }
    var email by remember { mutableStateOf("") }
    var age by remember { mutableStateOf("") }

    val isValid by remember {
        derivedStateOf {
            name.length >= 2 &&
            email.contains("@") &&
            age.toIntOrNull()?.let { it in 1..150 } ?: false
        }
    }

    Column(Modifier.padding(16.dp)) {
        OutlinedTextField(value = name, onValueChange = { name = it }, label = { Text("名前") }, modifier = Modifier.fillMaxWidth())
        OutlinedTextField(value = email, onValueChange = { email = it }, label = { Text("メール") }, modifier = Modifier.fillMaxWidth())
        OutlinedTextField(value = age, onValueChange = { age = it }, label = { Text("年齢") }, modifier = Modifier.fillMaxWidth())
        Spacer(Modifier.height(16.dp))
        Button(onClick = {}, enabled = isValid, modifier = Modifier.fillMaxWidth()) { Text("送信") }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `derivedStateOf` | 派生状態の計算 |
| `remember` | 計算結果のキャッシュ |
| `by` delegate | 値の自動取得 |
| 条件判定 | 閾値ベースの判定 |

- `derivedStateOf`で入力状態から派生する値を計算
- 入力が変わらない限り再計算しない（メモ化）
- スクロール位置やフォームバリデーションに最適
- `remember { derivedStateOf {} }`のパターンで使用

---

8種類のAndroidアプリテンプレート（パフォーマンス最適化済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [remember/key](https://zenn.dev/myougatheaxo/articles/android-compose-compose-remember-key-2026)
- [State Hoisting](https://zenn.dev/myougatheaxo/articles/android-compose-compose-state-hoisting-2026)
- [Stable Annotation](https://zenn.dev/myougatheaxo/articles/android-compose-compose-stable-annotation-2026)
