---
title: "Compose InAppPurchase完全ガイド — Google Play課金/サブスク/購入フロー"
emoji: "💰"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "billing"]
published: true
---

## この記事で学べること

**Compose InAppPurchase**（Google Play Billing、一回限り購入、サブスクリプション、購入復元）を解説します。

---

## セットアップ

```groovy
// build.gradle
dependencies {
    implementation("com.android.billingclient:billing-ktx:7.0.0")
}
```

---

## BillingClient初期化

```kotlin
class BillingManager(private val activity: Activity) {
    private var billingClient: BillingClient? = null
    private val _products = MutableStateFlow<List<ProductDetails>>(emptyList())
    val products: StateFlow<List<ProductDetails>> = _products

    fun initialize() {
        billingClient = BillingClient.newBuilder(activity)
            .setListener { billingResult, purchases ->
                if (billingResult.responseCode == BillingClient.BillingResponseCode.OK) {
                    purchases?.forEach { handlePurchase(it) }
                }
            }
            .enablePendingPurchases()
            .build()

        billingClient?.startConnection(object : BillingClientStateListener {
            override fun onBillingSetupFinished(result: BillingResult) {
                if (result.responseCode == BillingClient.BillingResponseCode.OK) {
                    queryProducts()
                }
            }
            override fun onBillingServiceDisconnected() { /* 再接続処理 */ }
        })
    }

    private fun queryProducts() {
        val params = QueryProductDetailsParams.newBuilder()
            .setProductList(listOf(
                QueryProductDetailsParams.Product.newBuilder()
                    .setProductId("premium_upgrade")
                    .setProductType(BillingClient.ProductType.INAPP)
                    .build()
            )).build()
        billingClient?.queryProductDetailsAsync(params) { _, details ->
            _products.value = details
        }
    }

    fun purchase(productDetails: ProductDetails) {
        val params = BillingFlowParams.newBuilder()
            .setProductDetailsParamsList(listOf(
                BillingFlowParams.ProductDetailsParams.newBuilder()
                    .setProductDetails(productDetails)
                    .build()
            )).build()
        billingClient?.launchBillingFlow(activity, params)
    }

    private fun handlePurchase(purchase: Purchase) {
        if (purchase.purchaseState == Purchase.PurchaseState.PURCHASED && !purchase.isAcknowledged) {
            val params = AcknowledgePurchaseParams.newBuilder()
                .setPurchaseToken(purchase.purchaseToken).build()
            billingClient?.acknowledgePurchase(params) { /* 確認完了 */ }
        }
    }
}
```

---

## Compose UI

```kotlin
@Composable
fun PurchaseScreen(billingManager: BillingManager) {
    val products by billingManager.products.collectAsStateWithLifecycle()

    LazyColumn(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        items(products) { product ->
            ElevatedCard(Modifier.fillMaxWidth()) {
                Column(Modifier.padding(16.dp)) {
                    Text(product.name, style = MaterialTheme.typography.titleMedium)
                    Text(product.description, style = MaterialTheme.typography.bodyMedium)
                    Spacer(Modifier.height(8.dp))
                    val price = product.oneTimePurchaseOfferDetails?.formattedPrice ?: ""
                    Button(onClick = { billingManager.purchase(product) },
                        Modifier.fillMaxWidth()) {
                        Text("$price で購入")
                    }
                }
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `BillingClient` | 課金クライアント |
| `ProductDetails` | 商品情報 |
| `BillingFlowParams` | 購入フロー |
| `acknowledgePurchase` | 購入確認（必須） |

- `BillingClient`でGoogle Play課金APIに接続
- `queryProductDetailsAsync`で商品情報を取得
- `launchBillingFlow`で購入ダイアログを表示
- 購入後は必ず`acknowledgePurchase`を呼ぶ（3日以内）

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose InAppReview](https://zenn.dev/myougatheaxo/articles/android-compose-compose-in-app-review-2026)
- [Compose InAppUpdate](https://zenn.dev/myougatheaxo/articles/android-compose-compose-in-app-update-2026)
- [Compose PlayIntegrity](https://zenn.dev/myougatheaxo/articles/android-compose-compose-play-integrity-2026)
