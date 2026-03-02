---
title: "センサーAPI完全ガイド — 加速度/ジャイロ/歩数計/Compose連携"
emoji: "📡"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "sensor"]
published: true
---

## この記事で学べること

**センサーAPI**（加速度センサー、ジャイロスコープ、歩数計、近接センサー、Compose連携）を解説します。

---

## センサーをFlowに変換

```kotlin
fun sensorFlow(
    context: Context,
    sensorType: Int,
    samplingPeriod: Int = SensorManager.SENSOR_DELAY_UI
): Flow<FloatArray> = callbackFlow {
    val sensorManager = context.getSystemService(Context.SENSOR_SERVICE) as SensorManager
    val sensor = sensorManager.getDefaultSensor(sensorType) ?: run {
        close(IllegalStateException("Sensor not available"))
        return@callbackFlow
    }

    val listener = object : SensorEventListener {
        override fun onSensorChanged(event: SensorEvent) {
            trySend(event.values.copyOf())
        }
        override fun onAccuracyChanged(sensor: Sensor?, accuracy: Int) {}
    }

    sensorManager.registerListener(listener, sensor, samplingPeriod)
    awaitClose { sensorManager.unregisterListener(listener) }
}
```

---

## 加速度センサー

```kotlin
@Composable
fun AccelerometerScreen() {
    val context = LocalContext.current
    var acceleration by remember { mutableStateOf(floatArrayOf(0f, 0f, 0f)) }

    LaunchedEffect(Unit) {
        sensorFlow(context, Sensor.TYPE_ACCELEROMETER)
            .collect { values -> acceleration = values }
    }

    Column(Modifier.fillMaxSize().padding(16.dp)) {
        Text("加速度センサー", style = MaterialTheme.typography.headlineMedium)
        Spacer(Modifier.height(16.dp))
        Text("X: ${"%.2f".format(acceleration[0])} m/s²")
        Text("Y: ${"%.2f".format(acceleration[1])} m/s²")
        Text("Z: ${"%.2f".format(acceleration[2])} m/s²")

        Spacer(Modifier.height(24.dp))
        Canvas(Modifier.size(200.dp)) {
            val centerX = size.width / 2
            val centerY = size.height / 2
            val ballX = centerX + acceleration[0] * -10
            val ballY = centerY + acceleration[1] * 10

            drawCircle(Color.LightGray, radius = size.minDimension / 2)
            drawCircle(Color.Red, radius = 20f, center = Offset(ballX, ballY))
        }
    }
}
```

---

## 歩数計

```kotlin
@Composable
fun StepCounterScreen() {
    val context = LocalContext.current
    var steps by remember { mutableIntStateOf(0) }

    LaunchedEffect(Unit) {
        sensorFlow(context, Sensor.TYPE_STEP_COUNTER)
            .collect { values -> steps = values[0].toInt() }
    }

    Column(
        Modifier.fillMaxSize().padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text("今日の歩数", style = MaterialTheme.typography.headlineMedium)
        Text("$steps", style = MaterialTheme.typography.displayLarge)
        Text("歩", style = MaterialTheme.typography.headlineSmall)

        Spacer(Modifier.height(24.dp))
        LinearProgressIndicator(
            progress = { minOf(steps / 10000f, 1f) },
            modifier = Modifier.fillMaxWidth()
        )
        Text("目標: 10,000歩", style = MaterialTheme.typography.bodySmall)
    }
}
```

---

## シェイク検出

```kotlin
@Composable
fun ShakeDetector(onShake: () -> Unit) {
    val context = LocalContext.current
    var lastShakeTime by remember { mutableLongStateOf(0L) }

    LaunchedEffect(Unit) {
        sensorFlow(context, Sensor.TYPE_ACCELEROMETER)
            .collect { values ->
                val magnitude = sqrt(values[0] * values[0] + values[1] * values[1] + values[2] * values[2])
                if (magnitude > 15f) {
                    val now = System.currentTimeMillis()
                    if (now - lastShakeTime > 1000) {
                        lastShakeTime = now
                        onShake()
                    }
                }
            }
    }
}
```

---

## まとめ

| センサー | TYPE |
|---------|------|
| 加速度 | `TYPE_ACCELEROMETER` |
| ジャイロ | `TYPE_GYROSCOPE` |
| 歩数 | `TYPE_STEP_COUNTER` |
| 近接 | `TYPE_PROXIMITY` |
| 光量 | `TYPE_LIGHT` |

- `callbackFlow`でセンサーをFlow化
- `SensorManager`でセンサー登録/解除
- シェイク検出は加速度の大きさで判定
- `awaitClose`で確実にリスナー解除

---

8種類のAndroidアプリテンプレート（センサー連携対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Health Connect](https://zenn.dev/myougatheaxo/articles/android-compose-health-connect-2026)
- [位置情報](https://zenn.dev/myougatheaxo/articles/android-compose-location-geofence-2026)
- [Bluetooth BLE](https://zenn.dev/myougatheaxo/articles/android-compose-bluetooth-ble-2026)
