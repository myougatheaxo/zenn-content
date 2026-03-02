---
title: "Effect Handler完全ガイド — LaunchedEffect/DisposableEffect/SideEffect/produceState"
emoji: "⚡"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "lifecycle"]
published: true
---

## この記事で学べること

**Effect Handler**（LaunchedEffect、DisposableEffect、SideEffect、produceState、snapshotFlow）を解説します。

---

## LaunchedEffect

```kotlin
@Composable
fun LaunchedEffectExample(userId: Long) {
    var user by remember { mutableStateOf<User?>(null) }

    // userIdが変わるたびに再起動
    LaunchedEffect(userId) {
        user = repository.getUser(userId)
    }

    // 一回だけ（Unit key）
    LaunchedEffect(Unit) {
        delay(3000)
        // スプラッシュ後の処理
    }

    user?.let { Text(it.name) }
}
```

---

## DisposableEffect

```kotlin
@Composable
fun LifecycleObserverExample() {
    val lifecycleOwner = LocalLifecycleOwner.current

    DisposableEffect(lifecycleOwner) {
        val observer = LifecycleEventObserver { _, event ->
            when (event) {
                Lifecycle.Event.ON_RESUME -> { /* 再開処理 */ }
                Lifecycle.Event.ON_PAUSE -> { /* 一時停止処理 */ }
                else -> {}
            }
        }
        lifecycleOwner.lifecycle.addObserver(observer)
        onDispose {
            lifecycleOwner.lifecycle.removeObserver(observer)
        }
    }
}

// センサーリスナー
@Composable
fun SensorExample() {
    val context = LocalContext.current

    DisposableEffect(Unit) {
        val sensorManager = context.getSystemService(Context.SENSOR_SERVICE) as SensorManager
        val sensor = sensorManager.getDefaultSensor(Sensor.TYPE_ACCELEROMETER)
        val listener = object : SensorEventListener {
            override fun onSensorChanged(event: SensorEvent?) {}
            override fun onAccuracyChanged(sensor: Sensor?, accuracy: Int) {}
        }
        sensorManager.registerListener(listener, sensor, SensorManager.SENSOR_DELAY_NORMAL)
        onDispose { sensorManager.unregisterListener(listener) }
    }
}
```

---

## produceState / snapshotFlow

```kotlin
@Composable
fun ProduceStateExample(url: String): State<Result<String>> {
    return produceState<Result<String>>(initialValue = Result.Loading, url) {
        value = try {
            val data = api.fetch(url)
            Result.Success(data)
        } catch (e: Exception) {
            Result.Error(e)
        }
    }
}

@Composable
fun SnapshotFlowExample(listState: LazyListState) {
    LaunchedEffect(listState) {
        snapshotFlow { listState.firstVisibleItemIndex }
            .distinctUntilChanged()
            .collect { index ->
                analytics.logScroll(index)
            }
    }
}
```

---

## まとめ

| Effect | 用途 |
|--------|------|
| `LaunchedEffect` | コルーチン起動 |
| `DisposableEffect` | リソース解放 |
| `SideEffect` | 毎コンポーズ時 |
| `produceState` | 非同期→State変換 |
| `snapshotFlow` | State→Flow変換 |

- `LaunchedEffect(key)`でkeyが変わると再起動
- `DisposableEffect`で`onDispose`によるクリーンアップ
- `produceState`で非同期結果をStateに変換
- `snapshotFlow`でCompose StateをFlowに変換

---

8種類のAndroidアプリテンプレート（アーキテクチャ実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Side Effects](https://zenn.dev/myougatheaxo/articles/android-compose-compose-side-effects-2026)
- [State Management](https://zenn.dev/myougatheaxo/articles/android-compose-compose-state-management-2026)
- [Coroutine](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-exception-2026)
