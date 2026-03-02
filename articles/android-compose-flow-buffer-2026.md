---
title: "Flow buffer完全ガイド — バッファリング/conflate/collectLatest/バックプレッシャー"
emoji: "📦"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "flow"]
published: true
---

## この記事で学べること

**Flow buffer**（buffer、conflate、collectLatest、バックプレッシャー制御）を解説します。

---

## buffer基本

```kotlin
// バッファなし: emit→collect→emit→collect（直列）
flow {
    repeat(5) { emit(it); delay(100) } // 生産: 100ms間隔
}.collect { value ->
    delay(300) // 消費: 300ms
    println(value)
}
// 合計: 5 * (100+300) = 2000ms

// buffer付き: emitとcollectが並行動作
flow {
    repeat(5) { emit(it); delay(100) }
}.buffer()
.collect { value ->
    delay(300)
    println(value)
}
// 合計: 100*5 + 300 ≈ 800ms（高速化）
```

---

## conflate（最新値のみ保持）

```kotlin
// conflate: 処理が追いつかない場合、中間値をスキップ
flow {
    repeat(10) { emit(it); delay(100) }
}.conflate()
.collect { value ->
    delay(300) // 300ms処理
    println(value)
}
// 0, 2, 5, 8 など（中間値がスキップされる）

// 実用: センサーデータ
@Composable
fun SensorDataScreen(viewModel: SensorViewModel = hiltViewModel()) {
    val data by viewModel.sensorData.collectAsStateWithLifecycle()

    // ViewModelで
    // sensorFlow.conflate().collect { _sensorData.value = it }
    // → 最新値のみUI更新

    Text("加速度: ${data.x}, ${data.y}, ${data.z}")
}
```

---

## collectLatest

```kotlin
// collectLatest: 新しい値が来たら前の処理をキャンセル
flow {
    emit("検索A"); delay(100)
    emit("検索B"); delay(100)
    emit("検索C")
}.collectLatest { query ->
    delay(500) // 検索処理
    println("結果: $query")
}
// 結果: 検索C のみ出力（A,Bはキャンセル）

// 実用: リアルタイム検索
@HiltViewModel
class SearchViewModel @Inject constructor(
    private val repository: SearchRepository
) : ViewModel() {
    private val _query = MutableStateFlow("")
    private val _results = MutableStateFlow<List<SearchResult>>(emptyList())
    val results: StateFlow<List<SearchResult>> = _results

    init {
        viewModelScope.launch {
            _query.debounce(300).collectLatest { query ->
                if (query.length >= 2) {
                    _results.value = repository.search(query)
                }
            }
        }
    }
}
```

---

## まとめ

| 演算子 | 動作 |
|--------|------|
| `buffer()` | 生産と消費を並行化 |
| `conflate()` | 最新値のみ保持 |
| `collectLatest` | 新しい値で前をキャンセル |
| `buffer(CONFLATED)` | conflateと同等 |

- `buffer()`で生産と消費のパイプライン化
- `conflate()`は高頻度データ（センサー等）に最適
- `collectLatest`は検索/フィルタに最適
- バックプレッシャーの制御でメモリとパフォーマンスを最適化

---

8種類のAndroidアプリテンプレート（Flow対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Flow zip](https://zenn.dev/myougatheaxo/articles/android-compose-flow-zip-2026)
- [Flow flatMap](https://zenn.dev/myougatheaxo/articles/android-compose-flow-flatmap-2026)
- [Flow Debounce](https://zenn.dev/myougatheaxo/articles/android-compose-flow-debounce-2026)
