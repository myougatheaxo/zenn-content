---
title: "BroadcastReceiver完全ガイド — システムイベント/カスタムブロードキャスト/Compose"
emoji: "📻"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "system"]
published: true
---

## この記事で学べること

**BroadcastReceiver**（システムイベント受信、カスタムブロードキャスト、動的登録、Compose統合）を解説します。

---

## システムブロードキャスト

```kotlin
class BootReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        if (intent.action == Intent.ACTION_BOOT_COMPLETED) {
            // 端末起動時の処理
            WorkManager.getInstance(context)
                .enqueue(OneTimeWorkRequestBuilder<SyncWorker>().build())
        }
    }
}

class BatteryReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        when (intent.action) {
            Intent.ACTION_BATTERY_LOW -> { /* バッテリー低下 */ }
            Intent.ACTION_POWER_CONNECTED -> { /* 充電開始 */ }
            Intent.ACTION_POWER_DISCONNECTED -> { /* 充電終了 */ }
        }
    }
}
```

```xml
<!-- AndroidManifest.xml -->
<receiver android:name=".BootReceiver"
    android:exported="false">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED" />
    </intent-filter>
</receiver>
```

---

## Compose動的登録

```kotlin
@Composable
fun BatteryStatus() {
    val context = LocalContext.current
    var batteryLevel by remember { mutableIntStateOf(0) }
    var isCharging by remember { mutableStateOf(false) }

    DisposableEffect(Unit) {
        val receiver = object : BroadcastReceiver() {
            override fun onReceive(ctx: Context, intent: Intent) {
                val level = intent.getIntExtra(BatteryManager.EXTRA_LEVEL, -1)
                val scale = intent.getIntExtra(BatteryManager.EXTRA_SCALE, -1)
                batteryLevel = (level * 100 / scale.toFloat()).toInt()

                val status = intent.getIntExtra(BatteryManager.EXTRA_STATUS, -1)
                isCharging = status == BatteryManager.BATTERY_STATUS_CHARGING
            }
        }

        val filter = IntentFilter(Intent.ACTION_BATTERY_CHANGED)
        context.registerReceiver(receiver, filter)

        onDispose { context.unregisterReceiver(receiver) }
    }

    Row(verticalAlignment = Alignment.CenterVertically) {
        Icon(
            if (isCharging) Icons.Default.BatteryChargingFull else Icons.Default.Battery5Bar,
            null
        )
        Text("$batteryLevel%")
    }
}
```

---

## Flow化

```kotlin
fun Context.broadcastFlow(vararg actions: String): Flow<Intent> = callbackFlow {
    val receiver = object : BroadcastReceiver() {
        override fun onReceive(context: Context, intent: Intent) {
            trySend(intent)
        }
    }

    val filter = IntentFilter().apply {
        actions.forEach { addAction(it) }
    }
    registerReceiver(receiver, filter)

    awaitClose { unregisterReceiver(receiver) }
}

// 使用
@Composable
fun AirplaneModeMonitor() {
    val context = LocalContext.current
    val airplaneMode by context
        .broadcastFlow(Intent.ACTION_AIRPLANE_MODE_CHANGED)
        .map { intent -> intent.getBooleanExtra("state", false) }
        .collectAsStateWithLifecycle(initialValue = false)

    if (airplaneMode) {
        Text("機内モード ON", color = MaterialTheme.colorScheme.error)
    }
}
```

---

## まとめ

| 登録方法 | ライフサイクル |
|----------|---------------|
| Manifest | アプリ未起動でも受信 |
| 動的登録 | Context存在中のみ |
| Flow化 | Compose連携 |

- Manifest登録で起動完了やインストール完了を受信
- `DisposableEffect`で動的登録/解除
- `callbackFlow`でFlowに変換しCompose連携
- Android 14+は一部ブロードキャストに`RECEIVER_NOT_EXPORTED`必須

---

8種類のAndroidアプリテンプレート（システム連携対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [WorkManager](https://zenn.dev/myougatheaxo/articles/android-compose-workmanager-2026)
- [Foreground Service](https://zenn.dev/myougatheaxo/articles/android-compose-foreground-service-2026)
- [アラーム](https://zenn.dev/myougatheaxo/articles/android-compose-alarm-scheduler-2026)
