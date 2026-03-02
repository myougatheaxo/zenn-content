---
title: "Kotlin Flow演算子ガイド — map/combine/debounce/flatMapLatest"
emoji: "🌊"
type: "tech"
topics: ["android", "kotlin", "coroutine", "flow"]
published: true
---

## この記事で学べること

Kotlin **Flow**の演算子（変換、結合、フィルタ、時間制御）を解説します。

---

## 変換演算子

```kotlin
// map: 要素を変換
val userNames: Flow<String> = userFlow.map { it.name }

// mapNotNull: nullを除外して変換
val activeUsers: Flow<User> = userFlow.mapNotNull { user ->
    user.takeIf { it.isActive }
}

// transform: 複数の値をemit
val searchResults: Flow<SearchResult> = queryFlow.transform { query ->
    emit(SearchResult.Loading)
    val results = repository.search(query)
    emit(SearchResult.Success(results))
}
```

---

## フィルタ演算子

```kotlin
// filter
val adults: Flow<User> = userFlow.filter { it.age >= 18 }

// distinctUntilChanged: 値が変わった時のみemit
val uniqueQueries: Flow<String> = queryFlow.distinctUntilChanged()

// take: 最初のN個
val firstThree: Flow<User> = userFlow.take(3)

// drop: 最初のN個をスキップ
val afterFirst: Flow<User> = userFlow.drop(1)
```

---

## 結合演算子

```kotlin
// combine: 最新の値同士を結合
val uiState: Flow<UiState> = combine(
    userFlow,
    settingsFlow,
    networkStatusFlow
) { user, settings, isOnline ->
    UiState(user = user, settings = settings, isOnline = isOnline)
}

// zip: 1対1で結合（遅い方に合わせる）
val pairs: Flow<Pair<User, Score>> = userFlow.zip(scoreFlow) { user, score ->
    Pair(user, score)
}

// merge: 複数Flowを1つに
val allEvents: Flow<Event> = merge(clickEvents, scrollEvents, timerEvents)
```

---

## 時間制御

```kotlin
// debounce: 一定時間入力がなければemit（検索用）
val debouncedQuery: Flow<String> = queryFlow
    .debounce(300) // 300ms

// sample: 一定間隔でサンプリング
val sampledLocation: Flow<Location> = locationFlow
    .sample(1000) // 1秒ごと

// timeout: タイムアウト
val withTimeout: Flow<Data> = dataFlow
    .timeout(5.seconds)
```

---

## flatMap系

```kotlin
// flatMapLatest: 新しい値が来たら前のFlowをキャンセル
val searchResults: Flow<List<Result>> = queryFlow
    .debounce(300)
    .distinctUntilChanged()
    .flatMapLatest { query ->
        if (query.isBlank()) flowOf(emptyList())
        else repository.search(query)
    }

// flatMapConcat: 順番に処理
val details: Flow<UserDetail> = userIdsFlow
    .flatMapConcat { id ->
        repository.getUserDetail(id)
    }

// flatMapMerge: 並行で処理
val allDetails: Flow<UserDetail> = userIdsFlow
    .flatMapMerge(concurrency = 4) { id ->
        repository.getUserDetail(id)
    }
```

---

## エラーハンドリング

```kotlin
// catch: エラーをキャッチ
val safeFlow: Flow<UiState> = dataFlow
    .map { UiState.Success(it) }
    .catch { e -> emit(UiState.Error(e.message ?: "Error")) }

// retry: リトライ
val resilientFlow: Flow<Data> = dataFlow
    .retry(3) { e ->
        e is IOException && run { delay(1000); true }
    }

// onEach: 副作用
val loggedFlow: Flow<Data> = dataFlow
    .onEach { Log.d("TAG", "Received: $it") }
    .onStart { Log.d("TAG", "Flow started") }
    .onCompletion { Log.d("TAG", "Flow completed") }
```

---

## まとめ

| カテゴリ | 演算子 | 用途 |
|---------|--------|------|
| 変換 | map, transform | 要素の変換 |
| フィルタ | filter, distinctUntilChanged | 条件フィルタ |
| 結合 | combine, zip, merge | 複数Flow結合 |
| 時間 | debounce, sample, timeout | レート制御 |
| flatMap | flatMapLatest, Concat, Merge | Flow内Flow |
| エラー | catch, retry | エラー処理 |

---

8種類のAndroidアプリテンプレート（Flow設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Flow+Lifecycle](https://zenn.dev/myougatheaxo/articles/android-compose-flow-lifecycle-2026)
- [コルーチン例外処理](https://zenn.dev/myougatheaxo/articles/kotlin-coroutine-exception-2026)
- [検索/フィルターUI](https://zenn.dev/myougatheaxo/articles/android-compose-search-filter-2026)
