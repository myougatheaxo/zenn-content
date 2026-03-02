---
title: "Flow combine/zip完全ガイド — combine/zip/merge/flatMapConcat"
emoji: "🔗"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "coroutine"]
published: true
---

## この記事で学べること

**Flow合成**（combine、zip、merge、flatMapConcat、flatMapLatest）を解説します。

---

## combine

```kotlin
@HiltViewModel
class DashboardViewModel @Inject constructor(
    private val userRepo: UserRepository,
    private val settingsRepo: SettingsRepository
) : ViewModel() {
    val dashboardState = combine(
        userRepo.getUser(),
        settingsRepo.getSettings(),
        userRepo.getNotificationCount()
    ) { user, settings, notifCount ->
        DashboardState(
            userName = user.name,
            isDarkMode = settings.isDarkMode,
            notificationCount = notifCount
        )
    }.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), DashboardState())
}

data class DashboardState(
    val userName: String = "",
    val isDarkMode: Boolean = false,
    val notificationCount: Int = 0
)
```

---

## zip

```kotlin
// zip: 両方の値が揃ったら発行（1:1対応）
fun fetchAndMatch(): Flow<Pair<User, Profile>> {
    val users = flow { emit(api.getUser()) }
    val profiles = flow { emit(api.getProfile()) }
    return users.zip(profiles) { user, profile -> Pair(user, profile) }
}
```

---

## merge / flatMapLatest

```kotlin
// merge: 複数Flowを結合（どちらが発行しても流れる）
val allEvents = merge(
    localDb.getEvents(),
    remoteApi.getEvents().catch { emitAll(flowOf(emptyList())) }
)

// flatMapLatest: 新しい値が来たら前の処理をキャンセル
class SearchViewModel @Inject constructor(
    private val repository: SearchRepository
) : ViewModel() {
    private val _query = MutableStateFlow("")

    val results = _query
        .debounce(300)
        .distinctUntilChanged()
        .flatMapLatest { query ->
            if (query.length < 2) flowOf(emptyList())
            else repository.search(query)
        }
        .stateIn(viewModelScope, SharingStarted.Lazily, emptyList())

    fun updateQuery(q: String) { _query.value = q }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `combine` | 最新値の組み合わせ |
| `zip` | 1:1ペアリング |
| `merge` | 複数Flow統合 |
| `flatMapLatest` | 最新変換のみ |

- `combine`で複数StateFlowの最新値を合成
- `zip`で2つのFlowを1:1対応で結合
- `merge`で複数ソースを単一Flowに統合
- `flatMapLatest`で検索クエリの最新結果のみ取得

---

8種類のAndroidアプリテンプレート（Flow実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Coroutine Channel](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-channel-2026)
- [Coroutine Exception](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-exception-2026)
- [Retrofit + Flow](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-flow-2026)
