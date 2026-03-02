---
title: "Compose BroadcastReceiver完全ガイド — ブロードキャスト受信/Compose統合/イベント監視"
emoji: "📡"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "broadcast"]
published: true
---

## この記事で学べること

**Compose BroadcastReceiver**（ブロードキャスト受信、Compose内登録、システムイベント監視）を解説します。

---

## Compose内ブロードキャスト受信

```kotlin
@Composable
fun <T> rememberBroadcastReceiver(
    intentFilter: IntentFilter,
    transform: (Intent) -> T,
    initial: T
): State<T> {
    val context = LocalContext.current
    val state = remember { mutableStateOf(initial) }

    DisposableEffect(intentFilter) {
        val receiver = object : BroadcastReceiver() {
            override fun onReceive(ctx: Context, intent: Intent) {
                state.value = transform(intent)
            }
        }
        context.registerReceiver(receiver, intentFilter)
        onDispose { context.unregisterReceiver(receiver) }
    }

    return state
}

// 使用例: 機内モード監視
@Composable
fun AirplaneModeMonitor() {
    val isAirplaneMode by rememberBroadcastReceiver(
        intentFilter = IntentFilter(Intent.ACTION_AIRPLANE_MODE_CHANGED),
        transform = { it.getBooleanExtra("state", false) },
        initial = false
    )

    if (isAirplaneMode) {
        Card(colors = CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.errorContainer)) {
            Text("機内モードが有効です", Modifier.padding(16.dp))
        }
    }
}
```

---

## 画面ON/OFF監視

```kotlin
@Composable
fun ScreenStateMonitor() {
    var isScreenOn by remember { mutableStateOf(true) }

    DisposableEffect(Unit) {
        val receiver = object : BroadcastReceiver() {
            override fun onReceive(ctx: Context, intent: Intent) {
                isScreenOn = intent.action == Intent.ACTION_SCREEN_ON
            }
        }
        val filter = IntentFilter().apply {
            addAction(Intent.ACTION_SCREEN_ON)
            addAction(Intent.ACTION_SCREEN_OFF)
        }
        // 注: SCREEN_ON/OFFは動的登録のみ
        onDispose {}
    }

    Text("画面: ${if (isScreenOn) "ON" else "OFF"}")
}
```

---

## タイムゾーン変更

```kotlin
@Composable
fun TimeZoneAware() {
    var timeZone by remember { mutableStateOf(TimeZone.getDefault().id) }
    val context = LocalContext.current

    DisposableEffect(Unit) {
        val receiver = object : BroadcastReceiver() {
            override fun onReceive(ctx: Context, intent: Intent) {
                timeZone = TimeZone.getDefault().id
            }
        }
        context.registerReceiver(receiver, IntentFilter(Intent.ACTION_TIMEZONE_CHANGED))
        onDispose { context.unregisterReceiver(receiver) }
    }

    Text("タイムゾーン: $timeZone")
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `registerReceiver` | 動的登録 |
| `unregisterReceiver` | 登録解除 |
| `IntentFilter` | 受信フィルタ |
| `DisposableEffect` | ライフサイクル管理 |

- `DisposableEffect`で確実に`unregisterReceiver`
- 汎用`rememberBroadcastReceiver`で再利用可能に
- Android 14+は一部ブロードキャストにEXPORTEDフラグ必要
- システムイベント（バッテリー、接続状態等）の監視に活用

---

8種類のAndroidアプリテンプレート（システム連携対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose BatteryState](https://zenn.dev/myougatheaxo/articles/android-compose-compose-battery-state-2026)
- [Compose Connectivity](https://zenn.dev/myougatheaxo/articles/android-compose-compose-connectivity-2026)
- [Compose Lifecycle](https://zenn.dev/myougatheaxo/articles/android-compose-compose-lifecycle-2026)
