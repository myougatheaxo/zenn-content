---
title: "Flow transform完全ガイド — transform/scan/runningFold/distinctUntilChanged"
emoji: "🔧"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "flow"]
published: true
---

## この記事で学べること

**Flow transform**（transform、scan、runningFold、distinctUntilChanged、カスタム変換）を解説します。

---

## transform基本

```kotlin
// transform: map+filterの柔軟版（複数emit可能）
val numbers = flowOf(1, 2, 3, 4, 5)

numbers.transform { value ->
    emit(value)              // 元の値
    if (value % 2 == 0) {
        emit(value * 10)     // 偶数のみ10倍も追加
    }
}.collect { println(it) }
// 1, 2, 20, 3, 4, 40, 5

// 実用例: APIレスポンスの変換
repository.getItems().transform { items ->
    emit(UiState.Loading)
    val processed = items.map { it.toUiModel() }
    emit(UiState.Success(processed))
}
```

---

## scan / runningFold

```kotlin
// scan: 累積値を毎回emit
val events = flowOf(1, 2, 3, 4, 5)
events.scan(0) { acc, value -> acc + value }
    .collect { println(it) }
// 0, 1, 3, 6, 10, 15

// 実用: チャット履歴の蓄積
@HiltViewModel
class ChatViewModel @Inject constructor(
    private val chatRepository: ChatRepository
) : ViewModel() {
    val messages: StateFlow<List<Message>> = chatRepository.newMessages()
        .scan(emptyList<Message>()) { accumulated, newMessage ->
            accumulated + newMessage
        }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())
}

// runningFold: scanと同じ（エイリアス）
events.runningFold(0) { acc, value -> acc + value }
```

---

## distinctUntilChanged

```kotlin
// 重複排除（連続する同じ値をスキップ）
val sensorData = flow {
    emit(20); emit(20); emit(21); emit(21); emit(20)
}
sensorData.distinctUntilChanged().collect { println(it) }
// 20, 21, 20

// カスタム比較
data class User(val id: String, val name: String, val lastSeen: Long)

repository.observeUser()
    .distinctUntilChanged { old, new ->
        old.name == new.name // 名前が同じなら同一とみなす
    }
    .collect { user ->
        // 名前が変わった時のみ更新
    }

// distinctUntilChangedBy: 特定フィールドで比較
repository.observeUser()
    .distinctUntilChangedBy { it.name }
    .collect { /* 名前変更時のみ */ }
```

---

## まとめ

| 演算子 | 用途 |
|--------|------|
| `transform` | 柔軟な変換（複数emit） |
| `scan` | 累積値を毎回発行 |
| `distinctUntilChanged` | 連続重複排除 |
| `distinctUntilChangedBy` | 特定キーで重複排除 |

- `transform`は`map`+`filter`を超える柔軟性
- `scan`でイベントストリームを状態に変換
- `distinctUntilChanged`でUIの無駄な更新を防止
- カスタム比較関数で「変更」の定義を制御

---

8種類のAndroidアプリテンプレート（Flow対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Flow zip](https://zenn.dev/myougatheaxo/articles/android-compose-flow-zip-2026)
- [Flow flatMap](https://zenn.dev/myougatheaxo/articles/android-compose-flow-flatmap-2026)
- [Flow Buffer](https://zenn.dev/myougatheaxo/articles/android-compose-flow-buffer-2026)
