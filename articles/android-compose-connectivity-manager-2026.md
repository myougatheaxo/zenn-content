---
title: "ネットワーク接続監視完全ガイド — ConnectivityManager/Flow/オフラインUI"
emoji: "📶"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "network"]
published: true
---

## この記事で学べること

**ネットワーク接続監視**（ConnectivityManager、接続状態Flow、オフラインバナー、接続種別判定）を解説します。

---

## NetworkMonitor

```kotlin
class NetworkMonitor @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val connectivityManager = context.getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager

    val isOnline: Flow<Boolean> = callbackFlow {
        val callback = object : ConnectivityManager.NetworkCallback() {
            override fun onAvailable(network: Network) { trySend(true) }
            override fun onLost(network: Network) { trySend(false) }
            override fun onUnavailable() { trySend(false) }
        }

        val request = NetworkRequest.Builder()
            .addCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET)
            .build()

        connectivityManager.registerNetworkCallback(request, callback)

        // 初期状態
        val currentNetwork = connectivityManager.activeNetwork
        val capabilities = connectivityManager.getNetworkCapabilities(currentNetwork)
        trySend(capabilities?.hasCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET) == true)

        awaitClose { connectivityManager.unregisterNetworkCallback(callback) }
    }.distinctUntilChanged()

    val connectionType: Flow<ConnectionType> = callbackFlow {
        val callback = object : ConnectivityManager.NetworkCallback() {
            override fun onCapabilitiesChanged(network: Network, capabilities: NetworkCapabilities) {
                val type = when {
                    capabilities.hasTransport(NetworkCapabilities.TRANSPORT_WIFI) -> ConnectionType.WIFI
                    capabilities.hasTransport(NetworkCapabilities.TRANSPORT_CELLULAR) -> ConnectionType.CELLULAR
                    capabilities.hasTransport(NetworkCapabilities.TRANSPORT_ETHERNET) -> ConnectionType.ETHERNET
                    else -> ConnectionType.UNKNOWN
                }
                trySend(type)
            }
            override fun onLost(network: Network) { trySend(ConnectionType.NONE) }
        }

        connectivityManager.registerDefaultNetworkCallback(callback)
        awaitClose { connectivityManager.unregisterNetworkCallback(callback) }
    }.distinctUntilChanged()
}

enum class ConnectionType { WIFI, CELLULAR, ETHERNET, UNKNOWN, NONE }
```

---

## オフラインバナー

```kotlin
@Composable
fun OfflineBanner(networkMonitor: NetworkMonitor) {
    val isOnline by networkMonitor.isOnline.collectAsStateWithLifecycle(initialValue = true)

    AnimatedVisibility(
        visible = !isOnline,
        enter = slideInVertically() + fadeIn(),
        exit = slideOutVertically() + fadeOut()
    ) {
        Surface(
            Modifier.fillMaxWidth(),
            color = MaterialTheme.colorScheme.errorContainer
        ) {
            Row(
                Modifier.padding(horizontal = 16.dp, vertical = 8.dp),
                verticalAlignment = Alignment.CenterVertically
            ) {
                Icon(Icons.Default.WifiOff, null, tint = MaterialTheme.colorScheme.onErrorContainer)
                Spacer(Modifier.width(8.dp))
                Text("オフライン", color = MaterialTheme.colorScheme.onErrorContainer)
            }
        }
    }
}
```

---

## アプリ全体での使用

```kotlin
@Composable
fun AppContent(networkMonitor: NetworkMonitor) {
    val isOnline by networkMonitor.isOnline.collectAsStateWithLifecycle(initialValue = true)

    Scaffold(
        topBar = {
            Column {
                TopAppBar(title = { Text("My App") })
                OfflineBanner(networkMonitor)
            }
        }
    ) { padding ->
        if (isOnline) {
            OnlineContent(Modifier.padding(padding))
        } else {
            OfflineContent(Modifier.padding(padding))
        }
    }
}
```

---

## まとめ

| 機能 | API |
|------|-----|
| 接続監視 | `NetworkCallback` |
| 接続種別 | `NetworkCapabilities` |
| Flow化 | `callbackFlow` |
| 初期状態 | `activeNetwork` |

- `callbackFlow`でネットワーク状態をリアクティブに監視
- `distinctUntilChanged`で重複通知を排除
- `AnimatedVisibility`でオフラインバナーをスムーズ表示
- WiFi/Cellular/Ethernetの種別判定

---

8種類のAndroidアプリテンプレート（ネットワーク監視対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [オフラインファースト](https://zenn.dev/myougatheaxo/articles/android-compose-offline-first-2026)
- [Retrofit/OkHttp](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-okhttp-2026)
- [エラーUI](https://zenn.dev/myougatheaxo/articles/android-compose-error-handling-ui-2026)
