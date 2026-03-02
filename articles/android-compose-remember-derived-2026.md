---
title: "remember/derivedStateOf/snapshotFlow使い分けガイド"
emoji: "🔄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "state"]
published: true
---

## この記事で学べること

Composeの**状態派生**（remember、derivedStateOf、snapshotFlow、produceState）を使い分けます。

---

## remember vs rememberSaveable

```kotlin
@Composable
fun RememberComparison() {
    // remember: リコンポジション間で保持（画面回転で消失）
    var count by remember { mutableIntStateOf(0) }

    // rememberSaveable: 画面回転・プロセスkillでも保持
    var text by rememberSaveable { mutableStateOf("") }

    // remember(key): keyが変わった時に再計算
    val sorted = remember(items) { items.sortedBy { it.name } }

    // rememberSaveable + Saver: カスタムオブジェクト保存
    var user by rememberSaveable(stateSaver = UserSaver) {
        mutableStateOf(User("", ""))
    }
}

val UserSaver = run {
    val nameKey = "name"
    val emailKey = "email"
    mapSaver(
        save = { mapOf(nameKey to it.name, emailKey to it.email) },
        restore = { User(it[nameKey] as String, it[emailKey] as String) }
    )
}
```

---

## derivedStateOf

```kotlin
@Composable
fun DerivedStateExample() {
    var items by remember { mutableStateOf(listOf<Item>()) }
    var filter by remember { mutableStateOf("") }

    // ✅ 結果が変わった時のみリコンポーズ
    val filteredItems by remember {
        derivedStateOf {
            items.filter { it.name.contains(filter, ignoreCase = true) }
        }
    }

    // ✅ スクロール位置からの派生状態
    val listState = rememberLazyListState()
    val showButton by remember {
        derivedStateOf { listState.firstVisibleItemIndex > 5 }
    }

    // ❌ derivedStateOf不要（入力がStateでない）
    // val doubled = remember(count) { count * 2 } // これで十分
}
```

---

## snapshotFlow

```kotlin
@Composable
fun SnapshotFlowExample() {
    val listState = rememberLazyListState()

    // Compose状態 → Flow変換
    LaunchedEffect(listState) {
        snapshotFlow { listState.firstVisibleItemIndex }
            .distinctUntilChanged()
            .collect { index ->
                // スクロール位置が変わった時の処理
                analytics.trackScroll(index)
            }
    }

    // debounce付きの入力監視
    var query by remember { mutableStateOf("") }

    LaunchedEffect(Unit) {
        snapshotFlow { query }
            .debounce(300)
            .distinctUntilChanged()
            .collect { q ->
                viewModel.search(q)
            }
    }
}
```

---

## produceState

```kotlin
@Composable
fun ProduceStateExample(userId: String) {
    // suspend関数の結果をState化
    val user by produceState<User?>(initialValue = null, userId) {
        value = repository.getUser(userId)
    }

    // Flowの結果をState化（collectAsState代替）
    val settings by produceState(initialValue = Settings(), key1 = Unit) {
        dataStore.data.collect { value = it }
    }

    user?.let { UserProfile(it) } ?: LoadingIndicator()
}
```

---

## 使い分け判定フロー

```kotlin
// Q: 状態から別の値を計算する必要がある？
//   → 入力が1つ: remember(key) { 計算 }
//   → 入力が複数のState + 結果が頻繁に同じ: derivedStateOf

// Q: Compose状態をFlowとして使いたい？
//   → snapshotFlow { state }

// Q: suspend/Flowの結果をCompose Stateにしたい？
//   → Flow: collectAsStateWithLifecycle
//   → suspend: produceState

// Q: 画面回転後も値を保持したい？
//   → rememberSaveable（プリミティブ/Parcelable）
//   → rememberSaveable + Saver（カスタム型）
```

---

## まとめ

| API | 用途 | 入力 | 出力 |
|-----|------|------|------|
| `remember(key)` | キー変更時に再計算 | 値 | 値 |
| `derivedStateOf` | State→State変換（最適化） | State | State |
| `snapshotFlow` | State→Flow変換 | State | Flow |
| `produceState` | suspend/Flow→State変換 | suspend | State |
| `rememberSaveable` | 画面回転でも保持 | 値 | 値 |

---

8種類のAndroidアプリテンプレート（状態管理最適化済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [rememberパターン](https://zenn.dev/myougatheaxo/articles/android-compose-remember-patterns-2026)
- [リコンポジション最適化](https://zenn.dev/myougatheaxo/articles/android-compose-recomposition-debug-2026)
- [Side Effectsガイド](https://zenn.dev/myougatheaxo/articles/android-compose-side-effects-2026)
