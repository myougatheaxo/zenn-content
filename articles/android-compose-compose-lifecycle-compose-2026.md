---
title: "Lifecycle Compose完全ガイド — LifecycleEventEffect/collectAsStateWithLifecycle"
emoji: "🔄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "lifecycle"]
published: true
---

## この記事で学べること

**Lifecycle Compose**（LifecycleEventEffect、collectAsStateWithLifecycle、Lifecycle対応）を解説します。

---

## LifecycleEventEffect

```kotlin
@Composable
fun LifecycleAwareScreen() {
    // ON_RESUMEで実行
    LifecycleEventEffect(Lifecycle.Event.ON_RESUME) {
        // データ再取得、センサー再開等
    }

    // ON_PAUSEで実行
    LifecycleEventEffect(Lifecycle.Event.ON_PAUSE) {
        // リソース解放、保存等
    }

    // ON_STARTで実行
    LifecycleEventEffect(Lifecycle.Event.ON_START) {
        // 位置情報更新開始等
    }

    Text("Lifecycle対応画面")
}
```

---

## collectAsStateWithLifecycle

```kotlin
@HiltViewModel
class UserViewModel @Inject constructor(
    repository: UserRepository
) : ViewModel() {
    val users = repository.getUsers()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())
}

@Composable
fun UserScreen(viewModel: UserViewModel = hiltViewModel()) {
    // Lifecycle-awareにFlowを収集（バックグラウンドで自動停止）
    val users by viewModel.users.collectAsStateWithLifecycle()

    // minActiveState指定
    val liveData by viewModel.users.collectAsStateWithLifecycle(
        minActiveState = Lifecycle.State.RESUMED
    )

    LazyColumn {
        items(users) { user ->
            ListItem(headlineContent = { Text(user.name) })
        }
    }
}
```

---

## currentStateAsState

```kotlin
@Composable
fun LifecycleStateDisplay() {
    val lifecycleOwner = LocalLifecycleOwner.current
    val lifecycleState by lifecycleOwner.lifecycle.currentStateAsState()

    Text("現在のライフサイクル: $lifecycleState")

    // 状態に応じた処理
    if (lifecycleState.isAtLeast(Lifecycle.State.RESUMED)) {
        Text("画面が表示中です", color = Color.Green)
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `LifecycleEventEffect` | Lifecycleイベント処理 |
| `collectAsStateWithLifecycle` | Lifecycle対応Flow収集 |
| `currentStateAsState` | 現在状態のState変換 |
| `minActiveState` | 最小アクティブ状態 |

- `collectAsStateWithLifecycle`でバックグラウンド時に自動停止
- `LifecycleEventEffect`でON_RESUME等のイベント処理
- `WhileSubscribed(5000)`で5秒猶予後に停止
- `collectAsState`より`collectAsStateWithLifecycle`推奨

---

8種類のAndroidアプリテンプレート（Lifecycle対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Effect Handler](https://zenn.dev/myougatheaxo/articles/android-compose-compose-effect-handler-2026)
- [Side Effects](https://zenn.dev/myougatheaxo/articles/android-compose-compose-side-effects-2026)
- [State Management](https://zenn.dev/myougatheaxo/articles/android-compose-compose-state-management-2026)
