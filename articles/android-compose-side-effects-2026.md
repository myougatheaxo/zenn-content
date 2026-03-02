---
title: "Compose Side Effects完全ガイド — LaunchedEffect, DisposableEffect, SideEffectの使い分け"
emoji: "💫"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "sideeffect"]
published: true
---

## この記事で学べること

ComposeのSide Effect APIを正しく使い分ける方法を解説します。**いつ・どのEffectを使うべきか**が明確になります。

---

## LaunchedEffect（一度だけ実行）

```kotlin
@Composable
fun WelcomeScreen(viewModel: WelcomeViewModel) {
    LaunchedEffect(Unit) {
        viewModel.loadData()
    }

    // UI...
}
```

**key = Unit**: Compositionに入ったとき一度だけ実行。画面を離れると自動キャンセル。

### keyが変わると再実行

```kotlin
@Composable
fun UserProfile(userId: String) {
    LaunchedEffect(userId) {
        // userIdが変わるたびに再実行
        viewModel.loadUser(userId)
    }
}
```

---

## DisposableEffect（セットアップ + クリーンアップ）

```kotlin
@Composable
fun SensorScreen() {
    val context = LocalContext.current
    val sensorManager = remember {
        context.getSystemService<SensorManager>()
    }

    DisposableEffect(Unit) {
        val listener = object : SensorEventListener {
            override fun onSensorChanged(event: SensorEvent) { /* ... */ }
            override fun onAccuracyChanged(sensor: Sensor?, accuracy: Int) {}
        }

        val sensor = sensorManager?.getDefaultSensor(Sensor.TYPE_ACCELEROMETER)
        sensorManager?.registerListener(listener, sensor, SensorManager.SENSOR_DELAY_NORMAL)

        onDispose {
            sensorManager?.unregisterListener(listener)
        }
    }
}
```

`onDispose`で**リソースの解放**を行います。ライフサイクルオブザーバー、センサー、ブロードキャストレシーバーなどに最適。

---

## SideEffect（毎回のRecomposition）

```kotlin
@Composable
fun AnalyticsScreen(screenName: String) {
    val analytics = LocalAnalytics.current

    SideEffect {
        // Recompositionのたびに実行
        analytics.setCurrentScreen(screenName)
    }
}
```

`SideEffect`は**成功したComposition後に毎回**実行されます。外部システムとの同期に使用。

---

## rememberCoroutineScope

```kotlin
@Composable
fun ScrollToTopButton() {
    val scope = rememberCoroutineScope()
    val listState = rememberLazyListState()

    FloatingActionButton(onClick = {
        scope.launch {
            listState.animateScrollToItem(0)
        }
    }) {
        Icon(Icons.Default.ArrowUpward, "トップへ")
    }
}
```

**イベントハンドラ内**でCoroutineを起動するとき使います。LaunchedEffectはComposition時、rememberCoroutineScopeはユーザー操作時。

---

## rememberUpdatedState

```kotlin
@Composable
fun TimerEffect(onTimeout: () -> Unit) {
    val currentOnTimeout by rememberUpdatedState(onTimeout)

    LaunchedEffect(Unit) {
        delay(5000)
        currentOnTimeout()  // 最新のコールバックを呼ぶ
    }
}
```

LaunchedEffect内で**最新のラムダ**を参照したいときに使います。

---

## produceState

```kotlin
@Composable
fun UserCard(userId: String) {
    val user by produceState<User?>(initialValue = null, userId) {
        value = repository.getUser(userId)
    }

    user?.let {
        Text(it.name)
    } ?: CircularProgressIndicator()
}
```

suspend関数の結果をComposeのStateに変換。`collectAsState()`の汎用版。

---

## 使い分け早見表

| Effect | 用途 | key変更時 |
|--------|------|----------|
| `LaunchedEffect` | Coroutine起動（API呼び出し等） | キャンセル→再起動 |
| `DisposableEffect` | セットアップ + クリーンアップ | onDispose→再セットアップ |
| `SideEffect` | 外部システム同期 | 毎回実行 |
| `rememberCoroutineScope` | イベントハンドラ内Coroutine | - |
| `rememberUpdatedState` | 最新のラムダを参照 | - |
| `produceState` | suspend→State変換 | 再実行 |

---

## よくある間違い

```kotlin
// ❌ Composable内で直接Coroutine
@Composable
fun Bad() {
    val scope = rememberCoroutineScope()
    scope.launch { api.load() }  // Recompositionのたびに実行される！
}

// ✅ LaunchedEffectで一度だけ
@Composable
fun Good() {
    LaunchedEffect(Unit) {
        api.load()
    }
}
```

---

## まとめ

- **LaunchedEffect**: 一度だけCoroutine実行（API呼び出し、タイマー）
- **DisposableEffect**: リソースの登録/解除
- **SideEffect**: 外部システムとの同期
- **rememberCoroutineScope**: ボタンクリック等のイベント内
- **rememberUpdatedState**: Effect内で最新の値を参照
- **produceState**: 非同期結果をStateに変換

---

8種類のAndroidアプリテンプレート（正しいSide Effect実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Composeの状態管理入門](https://zenn.dev/myougatheaxo/articles/compose-state-management-2026)
- [Kotlin Coroutines & Flow入門](https://zenn.dev/myougatheaxo/articles/kotlin-coroutine-flow-2026)
- [ViewModel + StateFlow入門](https://zenn.dev/myougatheaxo/articles/android-viewmodel-stateflow-2026)
