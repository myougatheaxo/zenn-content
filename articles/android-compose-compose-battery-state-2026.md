---
title: "Compose BatteryState完全ガイド — バッテリー残量/充電状態/省電力モード"
emoji: "🔋"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "system"]
published: true
---

## この記事で学べること

**Compose BatteryState**（バッテリー残量取得、充電状態監視、省電力モード検出）を解説します。

---

## バッテリー状態取得

```kotlin
@Composable
fun BatteryInfo() {
    val context = LocalContext.current
    var batteryLevel by remember { mutableIntStateOf(0) }
    var isCharging by remember { mutableStateOf(false) }

    DisposableEffect(Unit) {
        val receiver = object : BroadcastReceiver() {
            override fun onReceive(ctx: Context, intent: Intent) {
                val level = intent.getIntExtra(BatteryManager.EXTRA_LEVEL, -1)
                val scale = intent.getIntExtra(BatteryManager.EXTRA_SCALE, -1)
                batteryLevel = (level * 100 / scale.coerceAtLeast(1))
                val status = intent.getIntExtra(BatteryManager.EXTRA_STATUS, -1)
                isCharging = status == BatteryManager.BATTERY_STATUS_CHARGING
                    || status == BatteryManager.BATTERY_STATUS_FULL
            }
        }
        context.registerReceiver(receiver, IntentFilter(Intent.ACTION_BATTERY_CHANGED))
        onDispose { context.unregisterReceiver(receiver) }
    }

    Card(Modifier.fillMaxWidth().padding(16.dp)) {
        Column(Modifier.padding(16.dp)) {
            Text("バッテリー", style = MaterialTheme.typography.titleMedium)
            LinearProgressIndicator(
                progress = { batteryLevel / 100f },
                modifier = Modifier.fillMaxWidth().padding(vertical = 8.dp),
                color = when {
                    batteryLevel > 50 -> Color(0xFF4CAF50)
                    batteryLevel > 20 -> Color(0xFFFF9800)
                    else -> Color(0xFFF44336)
                }
            )
            Text("$batteryLevel% ${if (isCharging) "⚡充電中" else ""}")
        }
    }
}
```

---

## 省電力モード検出

```kotlin
@Composable
fun PowerSaveAware(content: @Composable (isPowerSave: Boolean) -> Unit) {
    val context = LocalContext.current
    val powerManager = context.getSystemService(Context.POWER_SERVICE) as PowerManager
    var isPowerSave by remember { mutableStateOf(powerManager.isPowerSaveMode) }

    DisposableEffect(Unit) {
        val receiver = object : BroadcastReceiver() {
            override fun onReceive(ctx: Context, intent: Intent) {
                isPowerSave = powerManager.isPowerSaveMode
            }
        }
        context.registerReceiver(receiver, IntentFilter(PowerManager.ACTION_POWER_SAVE_MODE_CHANGED))
        onDispose { context.unregisterReceiver(receiver) }
    }

    content(isPowerSave)
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `ACTION_BATTERY_CHANGED` | バッテリー情報 |
| `BatteryManager` | レベル/充電状態 |
| `PowerManager` | 省電力モード |
| `BroadcastReceiver` | 状態変更監視 |

- `ACTION_BATTERY_CHANGED`はsticky broadcastで即座に情報取得
- `DisposableEffect`で`unregisterReceiver`を確実に
- 省電力モード時はアニメーション削減等のUX配慮
- バッテリー残量に応じたUI色変更でUX向上

---

8種類のAndroidアプリテンプレート（システム連携対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Connectivity](https://zenn.dev/myougatheaxo/articles/android-compose-compose-connectivity-2026)
- [Compose ForegroundService](https://zenn.dev/myougatheaxo/articles/android-compose-compose-foreground-service-2026)
- [WorkManager Basic](https://zenn.dev/myougatheaxo/articles/android-compose-workmanager-basic-2026)
