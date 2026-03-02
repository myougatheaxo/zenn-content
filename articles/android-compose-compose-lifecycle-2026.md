---
title: "Compose Lifecycle完全ガイド — LifecycleOwner/LifecycleEventEffect/状態復元"
emoji: "♻️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "lifecycle"]
published: true
---

## この記事で学べること

**Compose Lifecycle**（LifecycleEventEffect、LifecycleResumeEffect、collectAsStateWithLifecycle）を解説します。

---

## LifecycleEventEffect

```kotlin
@Composable
fun LifecycleAwareScreen(viewModel: MainViewModel = hiltViewModel()) {
    LifecycleEventEffect(Lifecycle.Event.ON_RESUME) {
        viewModel.refreshData()
    }

    LifecycleEventEffect(Lifecycle.Event.ON_PAUSE) {
        viewModel.saveState()
    }

    val data by viewModel.data.collectAsStateWithLifecycle()
    LazyColumn {
        items(data) { item -> Text(item.title, Modifier.padding(16.dp)) }
    }
}
```

---

## LifecycleResumeEffect

```kotlin
@Composable
fun SensorScreen() {
    val context = LocalContext.current

    LifecycleResumeEffect(Unit) {
        val sensorManager = context.getSystemService(Context.SENSOR_SERVICE) as SensorManager
        val sensor = sensorManager.getDefaultSensor(Sensor.TYPE_ACCELEROMETER)
        val listener = object : SensorEventListener {
            override fun onSensorChanged(event: SensorEvent) { /* 処理 */ }
            override fun onAccuracyChanged(sensor: Sensor?, accuracy: Int) {}
        }

        sensorManager.registerListener(listener, sensor, SensorManager.SENSOR_DELAY_NORMAL)

        onPauseOrDispose {
            sensorManager.unregisterListener(listener)
        }
    }

    Text("センサー監視中")
}
```

---

## collectAsStateWithLifecycle

```kotlin
@Composable
fun LifecycleAwareCollection(viewModel: DataViewModel = hiltViewModel()) {
    // ON_STARTED以上でのみ収集（推奨）
    val items by viewModel.items.collectAsStateWithLifecycle()

    // カスタムライフサイクル状態
    val notifications by viewModel.notifications.collectAsStateWithLifecycle(
        minActiveState = Lifecycle.State.RESUMED
    )

    Column {
        Text("アイテム: ${items.size}")
        Text("通知: ${notifications.size}")
    }
}

@HiltViewModel
class DataViewModel @Inject constructor(repo: DataRepository) : ViewModel() {
    val items = repo.getItems().stateIn(
        viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList()
    )
    val notifications = repo.getNotifications().stateIn(
        viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList()
    )
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `LifecycleEventEffect` | ライフサイクルイベント処理 |
| `LifecycleResumeEffect` | Resume/Pause対応 |
| `collectAsStateWithLifecycle` | ライフサイクル対応Flow収集 |
| `WhileSubscribed(5000)` | 5秒猶予付き停止 |

- `collectAsStateWithLifecycle`はFlow収集の標準（`collectAsState`より推奨）
- `LifecycleEventEffect`でイベント駆動処理
- `LifecycleResumeEffect`でリソースの登録/解除
- `WhileSubscribed(5000)`で設定変更時の不要な再取得を防止

---

8種類のAndroidアプリテンプレート（ライフサイクル対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose ProcessDeath](https://zenn.dev/myougatheaxo/articles/android-compose-compose-process-death-2026)
- [Compose RememberSaveable](https://zenn.dev/myougatheaxo/articles/android-compose-compose-remember-saveable-2026)
- [Flow StateFlow](https://zenn.dev/myougatheaxo/articles/android-compose-flow-state-flow-2026)
