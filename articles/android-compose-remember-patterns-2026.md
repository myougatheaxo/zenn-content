---
title: "remember完全ガイド — remember/rememberSaveable/derivedStateOfの使い分け"
emoji: "🧠"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "state"]
published: true
---

## この記事で学べること

Composeの**remember系API**の使い分けと最適なパフォーマンスのための使い方を解説します。

---

## remember

```kotlin
@Composable
fun RememberExample() {
    // リコンポジション時に値を保持
    var count by remember { mutableIntStateOf(0) }

    // 計算結果のキャッシュ
    val expensiveList = remember(items) {
        items.filter { it.isActive }.sortedBy { it.name }
    }

    // オブジェクトの保持（リコンポジションで再生成を防ぐ）
    val interactionSource = remember { MutableInteractionSource() }
}
```

---

## rememberSaveable

```kotlin
@Composable
fun RememberSaveableExample() {
    // 画面回転でも値が保持される
    var text by rememberSaveable { mutableStateOf("") }
    var count by rememberSaveable { mutableIntStateOf(0) }

    // カスタムSaverでデータクラスを保存
    var user by rememberSaveable(stateSaver = UserSaver) {
        mutableStateOf(User("", ""))
    }

    Column(Modifier.padding(16.dp)) {
        TextField(value = text, onValueChange = { text = it })
        Text("Count: $count")
        Button(onClick = { count++ }) { Text("増やす") }
    }
}

// カスタムSaver
val UserSaver = run {
    val nameKey = "name"
    val emailKey = "email"
    mapSaver(
        save = { mapOf(nameKey to it.name, emailKey to it.email) },
        restore = { User(it[nameKey] as String, it[emailKey] as String) }
    )
}

// listSaverパターン
val UserListSaver = listSaver(
    save = { listOf(it.name, it.email) },
    restore = { User(it[0] as String, it[1] as String) }
)
```

---

## derivedStateOf

```kotlin
@Composable
fun DerivedStateExample(items: List<Item>) {
    var searchQuery by remember { mutableStateOf("") }

    // searchQueryかitemsが変わったときだけ再計算
    val filteredItems by remember(items) {
        derivedStateOf {
            if (searchQuery.isBlank()) items
            else items.filter { it.name.contains(searchQuery, ignoreCase = true) }
        }
    }

    // スクロール位置に基づく表示制御
    val listState = rememberLazyListState()
    val showTopButton by remember {
        derivedStateOf { listState.firstVisibleItemIndex > 3 }
    }

    Column {
        TextField(value = searchQuery, onValueChange = { searchQuery = it })
        Text("${filteredItems.size}件")
        LazyColumn(state = listState) {
            items(filteredItems) { Text(it.name) }
        }
    }
}
```

---

## remember vs rememberSaveable

```kotlin
@Composable
fun ComparisonExample() {
    // remember: リコンポジション間で保持
    // 画面回転・プロセスキルで失われる
    var tempState by remember { mutableStateOf("一時的") }

    // rememberSaveable: savedStateに保存
    // 画面回転・プロセスキルでも復元
    var persistedState by rememberSaveable { mutableStateOf("永続的") }
}
```

### 使い分けの指針

| シナリオ | API |
|---------|-----|
| UI一時状態（アニメーション、ホバー） | `remember` |
| ユーザー入力値（テキスト、選択） | `rememberSaveable` |
| 計算結果キャッシュ | `remember(key)` |
| 頻繁に変わる状態からの派生 | `derivedStateOf` |
| オブジェクトの再生成防止 | `remember { }` |

---

## パフォーマンス最適化

```kotlin
@Composable
fun PerformanceExample(items: List<Item>) {
    // NG: 毎回ラムダが再生成される
    LazyColumn {
        items(items) { item ->
            ItemRow(item, onClick = { viewModel.select(item.id) }) // 毎回新しいラムダ
        }
    }

    // OK: rememberでラムダをキャッシュ
    LazyColumn {
        items(items) { item ->
            val onClick = remember(item.id) { { viewModel.select(item.id) } }
            ItemRow(item, onClick = onClick)
        }
    }

    // OK: key付きrememberで重い計算をキャッシュ
    items.forEach { item ->
        val formattedDate = remember(item.timestamp) {
            dateFormatter.format(item.timestamp)
        }
        Text(formattedDate)
    }
}
```

---

## まとめ

- `remember`: リコンポジション間の値保持
- `rememberSaveable`: 画面回転・プロセスキルにも対応
- `derivedStateOf`: 状態の派生計算（不要な再計算を防止）
- `remember(key)`: キーが変わった時だけ再計算
- ユーザー入力は`rememberSaveable`、一時状態は`remember`
- カスタムSaver: `mapSaver`/`listSaver`でデータクラス保存

---

8種類のAndroidアプリテンプレート（State管理最適化済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [State管理完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-state-management-2026)
- [Side Effectsガイド](https://zenn.dev/myougatheaxo/articles/android-compose-side-effects-2026)
- [Coroutines & Flowガイド](https://zenn.dev/myougatheaxo/articles/kotlin-coroutines-flow-2026)
