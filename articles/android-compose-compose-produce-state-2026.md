---
title: "produceState完全ガイド — 非同期データ→State変換/ローディング管理"
emoji: "🔄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "state"]
published: true
---

## この記事で学べること

**produceState**（非同期データのState変換、ローディング管理、エラー処理）を解説します。

---

## produceState基本

```kotlin
@Composable
fun UserProfile(userId: Long, repository: UserRepository) {
    val userState by produceState<Result<User>?>(initialValue = null, userId) {
        value = try {
            Result.success(repository.getUser(userId))
        } catch (e: Exception) {
            Result.failure(e)
        }
    }

    when {
        userState == null -> CircularProgressIndicator()
        userState!!.isSuccess -> {
            val user = userState!!.getOrThrow()
            Text("${user.name} (${user.email})")
        }
        userState!!.isFailure -> Text("エラー: ${userState!!.exceptionOrNull()?.message}")
    }
}
```

---

## UiState管理

```kotlin
sealed interface UiState<out T> {
    data object Loading : UiState<Nothing>
    data class Success<T>(val data: T) : UiState<T>
    data class Error(val message: String) : UiState<Nothing>
}

@Composable
fun <T> produceUiState(key: Any?, producer: suspend () -> T): State<UiState<T>> {
    return produceState<UiState<T>>(initialValue = UiState.Loading, key) {
        value = try {
            UiState.Success(producer())
        } catch (e: Exception) {
            UiState.Error(e.message ?: "Unknown error")
        }
    }
}

@Composable
fun ItemListScreen(repository: ItemRepository) {
    val state by produceUiState(Unit) { repository.getItems() }

    when (val s = state) {
        is UiState.Loading -> CircularProgressIndicator()
        is UiState.Success -> LazyColumn { items(s.data) { Text(it.title) } }
        is UiState.Error -> Text(s.message, color = Color.Red)
    }
}
```

---

## Flow収集

```kotlin
@Composable
fun FlowCollector(flow: Flow<List<Item>>) {
    val items by produceState(initialValue = emptyList<Item>()) {
        flow.collect { value = it }
    }

    LazyColumn {
        items(items) { item -> ListItem(headlineContent = { Text(item.title) }) }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `produceState` | 非同期→State変換 |
| `initialValue` | 初期値 |
| `value =` | State更新 |
| key | 再実行トリガー |

- `produceState`で非同期処理の結果をStateに変換
- `initialValue`でローディング状態を表現
- keyが変わると再実行
- Flow収集にも使用可能

---

8種類のAndroidアプリテンプレート（状態管理対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Side Effect](https://zenn.dev/myougatheaxo/articles/android-compose-compose-side-effect-2026)
- [derivedStateOf](https://zenn.dev/myougatheaxo/articles/android-compose-compose-derived-state-2026)
- [snapshotFlow](https://zenn.dev/myougatheaxo/articles/android-compose-compose-snapshot-2026)
