---
title: "Composeライフサイクル対応ガイド — DisposableEffect/LaunchedEffect"
emoji: "🔄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "lifecycle"]
published: true
---

## この記事で学べること

Composeの**ライフサイクル対応**（LaunchedEffect、DisposableEffect、LifecycleEventEffect）を解説します。

---

## LaunchedEffect

```kotlin
@Composable
fun TimerScreen() {
    var seconds by remember { mutableStateOf(0) }

    // keyが変わると再起動、コンポジション離脱でキャンセル
    LaunchedEffect(Unit) {
        while (true) {
            delay(1000)
            seconds++
        }
    }

    Text("経過: ${seconds}秒")
}

// key付き: userIdが変わるたびに再取得
@Composable
fun UserDetail(userId: String, viewModel: UserViewModel = hiltViewModel()) {
    LaunchedEffect(userId) {
        viewModel.loadUser(userId)
    }

    // UI...
}
```

---

## DisposableEffect

```kotlin
@Composable
fun SensorScreen() {
    val context = LocalContext.current

    DisposableEffect(Unit) {
        val sensorManager = context.getSystemService<SensorManager>()!!
        val accelerometer = sensorManager.getDefaultSensor(Sensor.TYPE_ACCELEROMETER)

        val listener = object : SensorEventListener {
            override fun onSensorChanged(event: SensorEvent) {
                // センサー値処理
            }
            override fun onAccuracyChanged(sensor: Sensor?, accuracy: Int) {}
        }

        sensorManager.registerListener(listener, accelerometer, SensorManager.SENSOR_DELAY_UI)

        onDispose {
            sensorManager.unregisterListener(listener)
        }
    }
}

// BroadcastReceiver
@Composable
fun BatteryStatus() {
    val context = LocalContext.current
    var batteryLevel by remember { mutableIntStateOf(0) }

    DisposableEffect(Unit) {
        val receiver = object : BroadcastReceiver() {
            override fun onReceive(ctx: Context, intent: Intent) {
                batteryLevel = intent.getIntExtra(BatteryManager.EXTRA_LEVEL, 0)
            }
        }
        context.registerReceiver(receiver, IntentFilter(Intent.ACTION_BATTERY_CHANGED))

        onDispose {
            context.unregisterReceiver(receiver)
        }
    }

    Text("バッテリー: $batteryLevel%")
}
```

---

## LifecycleEventEffect

```kotlin
@Composable
fun CameraPreview() {
    // onResume/onPauseでカメラ制御
    LifecycleEventEffect(Lifecycle.Event.ON_RESUME) {
        startCamera()
    }

    LifecycleEventEffect(Lifecycle.Event.ON_PAUSE) {
        stopCamera()
    }
}

// LifecycleResumeEffect (resume/pause ペア)
@Composable
fun AnalyticsTracker(screenName: String) {
    LifecycleResumeEffect(screenName) {
        analytics.logScreenView(screenName)

        onPauseOrDispose {
            analytics.logScreenExit(screenName)
        }
    }
}
```

---

## collectAsStateWithLifecycle

```kotlin
@Composable
fun UserListScreen(viewModel: UserViewModel = hiltViewModel()) {
    // ライフサイクルに連動してFlowを収集
    // バックグラウンドでは自動停止
    val users by viewModel.users.collectAsStateWithLifecycle()
    val isLoading by viewModel.isLoading.collectAsStateWithLifecycle()

    if (isLoading) {
        CircularProgressIndicator()
    } else {
        LazyColumn {
            items(users) { user ->
                UserRow(user)
            }
        }
    }
}

// ViewModel
@HiltViewModel
class UserViewModel @Inject constructor(
    repository: UserRepository
) : ViewModel() {

    val users: StateFlow<List<User>> = repository.observeUsers()
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = emptyList()
        )
}
```

---

## rememberUpdatedState

```kotlin
@Composable
fun DelayedCallback(onTimeout: () -> Unit) {
    // 最新のコールバックを保持
    val currentOnTimeout by rememberUpdatedState(onTimeout)

    LaunchedEffect(Unit) {
        delay(5000)
        currentOnTimeout() // 5秒後に最新のコールバックを呼ぶ
    }
}

// snapshotFlow: Compose状態→Flow変換
@Composable
fun ScrollTracker(listState: LazyListState) {
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

| Effect | 用途 | クリーンアップ |
|--------|------|--------------|
| `LaunchedEffect` | 非同期処理の起動 | 自動キャンセル |
| `DisposableEffect` | リソース登録/解除 | `onDispose` |
| `LifecycleEventEffect` | ライフサイクルイベント | 自動 |
| `LifecycleResumeEffect` | Resume/Pauseペア | `onPauseOrDispose` |

- `collectAsStateWithLifecycle`でバックグラウンド時の無駄な収集を防止
- `rememberUpdatedState`でEffects内の最新値参照
- `snapshotFlow`でCompose状態→Flow変換
- `SharingStarted.WhileSubscribed(5000)`で画面回転対応

---

8種類のAndroidアプリテンプレート（ライフサイクル対応済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Side Effects](https://zenn.dev/myougatheaxo/articles/android-compose-side-effects-2026)
- [Flow+Lifecycle](https://zenn.dev/myougatheaxo/articles/android-compose-flow-lifecycle-2026)
- [remember/derivedStateOf](https://zenn.dev/myougatheaxo/articles/android-compose-remember-derived-2026)
