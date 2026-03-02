---
title: "ネットワーク状態監視ガイド — ConnectivityManager + Flow"
emoji: "📡"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "network"]
published: true
---

## この記事で学べること

Composeでの**ネットワーク状態監視**（ConnectivityManager、Flow、オフラインバナー）を解説します。

---

## ネットワーク状態の監視

```kotlin
class NetworkMonitor(context: Context) {
    private val connectivityManager =
        context.getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager

    val isOnline: Flow<Boolean> = callbackFlow {
        val callback = object : ConnectivityManager.NetworkCallback() {
            override fun onAvailable(network: Network) { trySend(true) }
            override fun onLost(network: Network) { trySend(false) }
        }

        val request = NetworkRequest.Builder()
            .addCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET)
            .build()

        connectivityManager.registerNetworkCallback(request, callback)

        // 初期状態
        val currentNetwork = connectivityManager.activeNetwork
        val caps = connectivityManager.getNetworkCapabilities(currentNetwork)
        trySend(caps?.hasCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET) == true)

        awaitClose { connectivityManager.unregisterNetworkCallback(callback) }
    }.distinctUntilChanged()
}
```

---

## オフラインバナー

```kotlin
@Composable
fun OfflineBanner(isOnline: Boolean) {
    AnimatedVisibility(
        visible = !isOnline,
        enter = expandVertically() + fadeIn(),
        exit = shrinkVertically() + fadeOut()
    ) {
        Surface(
            color = MaterialTheme.colorScheme.error,
            modifier = Modifier.fillMaxWidth()
        ) {
            Row(
                Modifier.padding(8.dp),
                horizontalArrangement = Arrangement.Center,
                verticalAlignment = Alignment.CenterVertically
            ) {
                Icon(Icons.Default.WifiOff, null, tint = MaterialTheme.colorScheme.onError)
                Spacer(Modifier.width(8.dp))
                Text("オフラインです", color = MaterialTheme.colorScheme.onError)
            }
        }
    }
}

@Composable
fun AppWithOfflineBanner(networkMonitor: NetworkMonitor) {
    val isOnline by networkMonitor.isOnline.collectAsStateWithLifecycle(initialValue = true)

    Scaffold { padding ->
        Column(Modifier.padding(padding)) {
            OfflineBanner(isOnline)
            AppContent()
        }
    }
}
```

---

## ネットワーク復帰時の自動再読み込み

```kotlin
class DataViewModel(
    private val repository: DataRepository,
    private val networkMonitor: NetworkMonitor
) : ViewModel() {

    val data: StateFlow<UiState<List<Item>>> = networkMonitor.isOnline
        .filter { it } // オンラインになった時のみ
        .flatMapLatest { repository.getItems() }
        .map<List<Item>, UiState<List<Item>>> { UiState.Success(it) }
        .catch { emit(UiState.Error(it.message ?: "エラー")) }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), UiState.Loading)
}
```

---

## まとめ

- `ConnectivityManager.NetworkCallback`でリアルタイム監視
- `callbackFlow`でFlowに変換
- `distinctUntilChanged()`で重複除去
- `AnimatedVisibility`でオフラインバナーアニメーション
- `filter { it }`でオンライン復帰時に自動リロード
- アプリ全体のScaffoldに組み込んで常時表示

---

8種類のAndroidアプリテンプレート（オフライン対応設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [オフラインファーストガイド](https://zenn.dev/myougatheaxo/articles/android-offline-first-2026)
- [Flow + Lifecycleガイド](https://zenn.dev/myougatheaxo/articles/android-compose-flow-lifecycle-2026)
- [Retrofit通信ガイド](https://zenn.dev/myougatheaxo/articles/android-retrofit-network-2026)
