---
title: "Compose A/Bテスト完全ガイド — Firebase A/B Testing/実験設計/結果分析/Compose連携"
emoji: "🧪"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "firebase"]
published: true
---

## この記事で学べること

**Compose A/Bテスト**（Firebase A/B Testing、実験設計、バリアント切り替え、結果分析）を解説します。

---

## A/Bテスト基盤

```kotlin
class AbTestManager @Inject constructor() {
    private val remoteConfig = Firebase.remoteConfig
    private val analytics = Firebase.analytics

    suspend fun initialize() {
        remoteConfig.fetchAndActivate().await()
    }

    fun getVariant(experimentKey: String): String =
        remoteConfig.getString(experimentKey)

    fun trackExposure(experimentKey: String, variant: String) {
        analytics.logEvent("ab_exposure") {
            param("experiment", experimentKey)
            param("variant", variant)
        }
    }

    fun trackConversion(experimentKey: String, variant: String, value: Double = 1.0) {
        analytics.logEvent("ab_conversion") {
            param("experiment", experimentKey)
            param("variant", variant)
            param("value", value)
        }
    }
}
```

---

## Compose連携

```kotlin
@HiltViewModel
class CheckoutViewModel @Inject constructor(
    private val abTest: AbTestManager
) : ViewModel() {
    val buttonVariant = MutableStateFlow("control")

    init {
        viewModelScope.launch {
            abTest.initialize()
            val variant = abTest.getVariant("checkout_button_experiment")
            buttonVariant.value = variant
            abTest.trackExposure("checkout_button_experiment", variant)
        }
    }

    fun onPurchase() {
        abTest.trackConversion("checkout_button_experiment", buttonVariant.value)
    }
}

@Composable
fun CheckoutScreen(viewModel: CheckoutViewModel = hiltViewModel()) {
    val variant by viewModel.buttonVariant.collectAsStateWithLifecycle()

    Column(Modifier.fillMaxSize().padding(16.dp), verticalArrangement = Arrangement.Center) {
        Text("合計: ¥1,980", style = MaterialTheme.typography.headlineMedium)
        Spacer(Modifier.height(24.dp))

        when (variant) {
            "control" -> Button(
                onClick = { viewModel.onPurchase() },
                modifier = Modifier.fillMaxWidth()
            ) { Text("購入する") }

            "variant_a" -> Button(
                onClick = { viewModel.onPurchase() },
                modifier = Modifier.fillMaxWidth(),
                colors = ButtonDefaults.buttonColors(containerColor = Color(0xFF4CAF50))
            ) { Text("今すぐ購入 →") }

            "variant_b" -> ElevatedButton(
                onClick = { viewModel.onPurchase() },
                modifier = Modifier.fillMaxWidth()
            ) {
                Icon(Icons.Default.ShoppingCart, contentDescription = null)
                Spacer(Modifier.width(8.dp))
                Text("カートに追加して購入")
            }
        }
    }
}
```

---

## レイアウトA/Bテスト

```kotlin
@Composable
fun ProductListScreen(variant: String, products: List<Product>) {
    when (variant) {
        "grid" -> LazyVerticalGrid(
            columns = GridCells.Fixed(2),
            contentPadding = PaddingValues(16.dp),
            verticalArrangement = Arrangement.spacedBy(8.dp),
            horizontalArrangement = Arrangement.spacedBy(8.dp)
        ) {
            items(products) { ProductGridItem(it) }
        }

        "list" -> LazyColumn(contentPadding = PaddingValues(16.dp)) {
            items(products) { ProductListItem(it) }
        }
    }
}
```

---

## まとめ

| 機能 | 用途 |
|------|------|
| Firebase A/B Testing | 実験管理 |
| Remote Config | バリアント配信 |
| Analytics | コンバージョン計測 |
| `trackExposure` | 表示イベント |

- Firebase A/B TestingでRemote Config経由のバリアント配信
- `trackExposure`で実験表示を記録、`trackConversion`で成果を計測
- Compose UIをバリアントに応じて動的に切り替え
- Firebase Consoleで統計的有意差を確認

---

8種類のAndroidアプリテンプレート（Firebase対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose FeatureFlag](https://zenn.dev/myougatheaxo/articles/android-compose-compose-feature-flag-2026)
- [Compose RemoteConfig](https://zenn.dev/myougatheaxo/articles/android-compose-compose-remoteconfig-2026)
- [Compose FirebaseAnalytics](https://zenn.dev/myougatheaxo/articles/android-compose-compose-firebase-analytics-2026)
