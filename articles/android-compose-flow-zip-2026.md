---
title: "Flow zip完全ガイド — 複数Flow結合/zip/combine/merge違い"
emoji: "🔗"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "flow"]
published: true
---

## この記事で学べること

**Flow zip**（zip演算子、combine/merge/zipの違い、複数データソース結合パターン）を解説します。

---

## zip基本

```kotlin
// zip: 2つのFlowをペアで結合（両方が値を出すまで待つ）
val names = flowOf("太郎", "花子", "次郎")
val ages = flowOf(25, 30, 28)

names.zip(ages) { name, age ->
    "$name ($age歳)"
}.collect { println(it) }
// 太郎 (25歳), 花子 (30歳), 次郎 (28歳)
```

---

## zip vs combine vs merge

```kotlin
// zip: 1:1対応（遅い方に合わせる）
val flow1 = flow { emit(1); delay(100); emit(2); delay(100); emit(3) }
val flow2 = flow { emit("A"); delay(200); emit("B"); delay(200); emit("C") }

flow1.zip(flow2) { num, letter -> "$num-$letter" }.collect { println(it) }
// 1-A, 2-B, 3-C

// combine: 最新値の組み合わせ（どちらかが更新されるたび）
flow1.combine(flow2) { num, letter -> "$num-$letter" }.collect { println(it) }
// 1-A, 2-A, 2-B, 3-B, 3-C

// merge: 単に統合（型が同じ場合）
merge(
    flow { emit("Source1: data") },
    flow { emit("Source2: data") }
).collect { println(it) }
```

---

## 実践: 複数API結合

```kotlin
@HiltViewModel
class DashboardViewModel @Inject constructor(
    private val userRepository: UserRepository,
    private val statsRepository: StatsRepository
) : ViewModel() {

    val dashboardState: StateFlow<DashboardState> = userRepository.getUser()
        .zip(statsRepository.getStats()) { user, stats ->
            DashboardState(user = user, stats = stats)
        }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000),
            DashboardState())
}

data class DashboardState(
    val user: User? = null,
    val stats: Stats? = null
)

@Composable
fun DashboardScreen(viewModel: DashboardViewModel = hiltViewModel()) {
    val state by viewModel.dashboardState.collectAsStateWithLifecycle()

    Column(Modifier.padding(16.dp)) {
        state.user?.let { Text("ようこそ、${it.name}さん") }
        state.stats?.let {
            Text("合計: ${it.total}件")
            Text("今月: ${it.monthly}件")
        }
    }
}
```

---

## まとめ

| 演算子 | 動作 |
|--------|------|
| `zip` | 1:1ペア結合（同期） |
| `combine` | 最新値結合（どちらか更新時） |
| `merge` | 単純統合（型統一） |

- `zip`は両方のFlowが値を出すまで待つ
- `combine`はどちらかが更新されるたびに最新の組み合わせを発行
- `merge`は複数Flowの値をそのまま流す
- APIの並行呼び出しと結合には`zip`が最適

---

8種類のAndroidアプリテンプレート（Flow対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Flow Combine](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-flow-combine-2026)
- [Flow Transform](https://zenn.dev/myougatheaxo/articles/android-compose-flow-transform-2026)
- [Flow Buffer](https://zenn.dev/myougatheaxo/articles/android-compose-flow-buffer-2026)
