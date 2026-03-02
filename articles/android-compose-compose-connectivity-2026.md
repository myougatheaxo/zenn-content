---
title: "ネットワーク接続監視完全ガイド — ConnectivityManager/Flow/オフライン検知"
emoji: "📶"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "network"]
published: true
---

## この記事で学べること

**ネットワーク接続監視**（ConnectivityManager、Flow変換、オフラインバナー表示）を解説します。

---

## ConnectivityManager→Flow

```kotlin
class ConnectivityObserver @Inject constructor(
    @ApplicationContext private val context: Context
) {
    val isOnline: Flow<Boolean> = callbackFlow {
        val manager = context.getSystemService(ConnectivityManager::class.java)
        val callback = object : ConnectivityManager.NetworkCallback() {
            override fun onAvailable(network: Network) { trySend(true) }
            override fun onLost(network: Network) { trySend(false) }
            override fun onUnavailable() { trySend(false) }
        }

        val request = NetworkRequest.Builder()
            .addCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET)
            .build()
        manager.registerNetworkCallback(request, callback)

        // 初期状態
        val isConnected = manager.activeNetwork?.let {
            manager.getNetworkCapabilities(it)
                ?.hasCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET)
        } ?: false
        trySend(isConnected)

        awaitClose { manager.unregisterNetworkCallback(callback) }
    }.distinctUntilChanged()
}
```

---

## オフラインバナー

```kotlin
@Composable
fun NetworkAwareApp(connectivityObserver: ConnectivityObserver) {
    val isOnline by connectivityObserver.isOnline
        .collectAsStateWithLifecycle(initialValue = true)

    Column(Modifier.fillMaxSize()) {
        AnimatedVisibility(visible = !isOnline) {
            Surface(
                color = MaterialTheme.colorScheme.error,
                modifier = Modifier.fillMaxWidth()
            ) {
                Row(
                    Modifier.padding(8.dp),
                    verticalAlignment = Alignment.CenterVertically,
                    horizontalArrangement = Arrangement.Center
                ) {
                    Icon(Icons.Default.WifiOff, null, tint = Color.White)
                    Spacer(Modifier.width(8.dp))
                    Text("オフラインです", color = Color.White)
                }
            }
        }

        // メインコンテンツ
        MainContent()
    }
}
```

---

## ViewModel連携

```kotlin
@HiltViewModel
class MainViewModel @Inject constructor(
    private val connectivityObserver: ConnectivityObserver,
    private val repository: DataRepository
) : ViewModel() {
    val isOnline = connectivityObserver.isOnline
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), true)

    // オンライン復帰時に自動同期
    init {
        viewModelScope.launch {
            connectivityObserver.isOnline
                .filter { it }
                .collect { repository.syncPendingChanges() }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `ConnectivityManager` | ネットワーク状態取得 |
| `NetworkCallback` | 変更通知 |
| `callbackFlow` | コールバック→Flow |
| `distinctUntilChanged` | 重複排除 |

- `callbackFlow`でConnectivityManagerをFlow化
- `AnimatedVisibility`でオフラインバナー表示
- オンライン復帰時に自動同期トリガー
- `distinctUntilChanged`で重複イベント排除

---

8種類のAndroidアプリテンプレート（ネットワーク対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [オフラインファースト](https://zenn.dev/myougatheaxo/articles/android-compose-offline-first-2026)
- [Retrofit + Flow](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-flow-2026)
- [WorkManager](https://zenn.dev/myougatheaxo/articles/android-compose-workmanager-2026)
