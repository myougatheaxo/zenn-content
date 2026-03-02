---
title: "Composeの状態管理入門 — remember, mutableStateOf, derivedStateOfの全て"
emoji: "🧠"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "state"]
published: true
---

## この記事で学べること

Composeの核心は**状態管理**です。`remember`と`mutableStateOf`を正しく使えれば、Composeの80%は理解できたと言えます。

---

## remember + mutableStateOf（基本）

```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }

    Button(onClick = { count++ }) {
        Text("Count: $count")
    }
}
```

- `mutableStateOf(0)`: 値が変わるとComposeが再描画（recomposition）する特殊な変数
- `remember`: recomposition時に値をリセットしない（メモ化）

**`remember`なしだと**、ボタンを押すたびに`count`が0に戻ります。

---

## rememberSaveable（画面回転対応）

```kotlin
@Composable
fun SearchBar() {
    var query by rememberSaveable { mutableStateOf("") }

    OutlinedTextField(
        value = query,
        onValueChange = { query = it },
        label = { Text("検索") }
    )
}
```

`remember`は画面回転で値が消えます。`rememberSaveable`なら**プロセス再生成後も値が復元**されます。

| | remember | rememberSaveable |
|---|---------|-----------------|
| recomposition | 保持 | 保持 |
| 画面回転 | リセット | 保持 |
| プロセス再生成 | リセット | 保持 |

**使い分け**: ユーザーが入力した値は`rememberSaveable`、一時的な計算結果は`remember`。

---

## derivedStateOf（計算コスト削減）

```kotlin
@Composable
fun FilteredList(items: List<String>, query: String) {
    val filteredItems by remember(items, query) {
        derivedStateOf {
            items.filter { it.contains(query, ignoreCase = true) }
        }
    }

    LazyColumn {
        items(filteredItems) { item ->
            Text(item)
        }
    }
}
```

`derivedStateOf`は**依存する値が変わったときだけ**再計算します。大量データのフィルタリングやソートに最適。

---

## State Hoisting（状態の引き上げ）

```kotlin
// ❌ 状態がComposable内部にある（テストしにくい）
@Composable
fun BadCounter() {
    var count by remember { mutableStateOf(0) }
    Button(onClick = { count++ }) { Text("$count") }
}

// ✅ 状態を外部から受け取る（テスト可能）
@Composable
fun GoodCounter(
    count: Int,
    onIncrement: () -> Unit
) {
    Button(onClick = onIncrement) { Text("$count") }
}
```

**State Hoisting**のルール：
1. 状態は**最も低い共通の親**で管理する
2. Composableには**値とイベントコールバック**を渡す
3. ViewModelの状態は`collectAsState()`で購読する

---

## ViewModel vs ローカル状態

| データ | 管理場所 |
|--------|---------|
| DBのデータ一覧 | ViewModel (StateFlow) |
| フォーム入力値 | rememberSaveable |
| ダイアログの開閉 | remember |
| スクロール位置 | rememberLazyListState |
| アニメーション | animateFloatAsState |

**原則**: ビジネスロジックに関わるデータはViewModel、UIの一時的な状態はローカル。

---

## snapshotFlow（StateをFlowに変換）

```kotlin
val listState = rememberLazyListState()

LaunchedEffect(listState) {
    snapshotFlow { listState.firstVisibleItemIndex }
        .collect { index ->
            if (index > 5) {
                // スクロールが5行を超えたらFABを表示
                showFab = true
            }
        }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `remember` | recomposition間で値を保持 |
| `mutableStateOf` | 値変更で再描画をトリガー |
| `rememberSaveable` | 画面回転・プロセス再生成でも保持 |
| `derivedStateOf` | 計算コストの高い派生値 |
| State Hoisting | テスト可能・再利用可能な設計 |

---

8種類のAndroidアプリテンプレート（正しい状態管理パターン実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ViewModel + StateFlow入門](https://zenn.dev/myougatheaxo/articles/android-viewmodel-stateflow-2026)
- [Composeレイアウト入門](https://zenn.dev/myougatheaxo/articles/compose-layout-basics-2026)
- [Compose入力フォーム完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-forms-input-2026)
- [Compose Animation入門](https://zenn.dev/myougatheaxo/articles/compose-animation-basics-2026)
