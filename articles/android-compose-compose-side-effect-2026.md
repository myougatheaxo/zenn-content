---
title: "SideEffect完全ガイド — SideEffect/LaunchedEffect/DisposableEffect比較"
emoji: "⚡"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "sideeffect"]
published: true
---

## この記事で学べること

**SideEffect**（SideEffect、LaunchedEffect、DisposableEffect、rememberCoroutineScope）を比較解説します。

---

## SideEffect

```kotlin
@Composable
fun SideEffectExample(userId: String) {
    // 毎回のリコンポジションで実行（非Compose操作の同期）
    SideEffect {
        Analytics.setUserId(userId)
    }

    Text("User: $userId")
}
```

---

## LaunchedEffect

```kotlin
@Composable
fun LaunchedEffectExample(snackbarHostState: SnackbarHostState, message: String?) {
    // keyが変わった時のみ再実行
    LaunchedEffect(message) {
        message?.let {
            snackbarHostState.showSnackbar(it)
        }
    }
}

// 初回のみ実行
@Composable
fun OneTimeEffect() {
    LaunchedEffect(Unit) {
        // 初回Compositionでのみ実行
        delay(1000)
        // 初期化処理
    }
}

// 定期実行
@Composable
fun PeriodicEffect(onTick: () -> Unit) {
    LaunchedEffect(Unit) {
        while (true) {
            delay(1000)
            onTick()
        }
    }
}
```

---

## DisposableEffect

```kotlin
@Composable
fun DisposableEffectExample() {
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

// コールバック登録/解除
@Composable
fun SensorListener(onData: (Float) -> Unit) {
    val context = LocalContext.current

    DisposableEffect(Unit) {
        val sensorManager = context.getSystemService(Context.SENSOR_SERVICE) as SensorManager
        val sensor = sensorManager.getDefaultSensor(Sensor.TYPE_ACCELEROMETER)
        val listener = object : SensorEventListener {
            override fun onSensorChanged(event: SensorEvent) { onData(event.values[0]) }
            override fun onAccuracyChanged(sensor: Sensor?, accuracy: Int) {}
        }
        sensorManager.registerListener(listener, sensor, SensorManager.SENSOR_DELAY_NORMAL)

        onDispose { sensorManager.unregisterListener(listener) }
    }
}
```

---

## まとめ

| API | タイミング | 用途 |
|-----|----------|------|
| `SideEffect` | 毎リコンポジション | 非Compose同期 |
| `LaunchedEffect` | key変更時 | suspend処理 |
| `DisposableEffect` | key変更時+クリーンアップ | リスナー登録 |
| `rememberCoroutineScope` | イベント駆動 | onClick等から |

- `SideEffect`は毎回実行（Analytics等に最適）
- `LaunchedEffect`はsuspend関数を安全に実行
- `DisposableEffect`はリソース登録/解除のペア
- `rememberCoroutineScope`はイベントハンドラ用

---

8種類のAndroidアプリテンプレート（ライフサイクル対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Effect Handler](https://zenn.dev/myougatheaxo/articles/android-compose-compose-effect-handler-2026)
- [Lifecycle Compose](https://zenn.dev/myougatheaxo/articles/android-compose-compose-lifecycle-compose-2026)
- [CompositionLocal](https://zenn.dev/myougatheaxo/articles/android-compose-compose-composition-local-2026)
