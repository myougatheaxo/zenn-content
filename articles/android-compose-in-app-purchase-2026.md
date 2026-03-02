---
title: "In-App Purchase (課金) + Compose連携ガイド"
emoji: "💰"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "billing"]
published: true
---

## この記事で学べること

Google Play **Billing Library**を使ったアプリ内課金（消耗品/サブスクリプション）を解説します。

---

## セットアップ

```kotlin
dependencies {
    implementation("com.android.billingclient:billing-ktx:7.0.0")
}
```

---

## BillingManager

```kotlin
class BillingManager(private val activity: Activity) {
    private val billingClient = BillingClient.newBuilder(activity)
        .setListener { billingResult, purchases ->
            if (billingResult.responseCode == BillingClient.BillingResponseCode.OK) {
                purchases?.forEach { purchase ->
                    handlePurchase(purchase)
                }
            }
        }
        .enablePendingPurchases()
        .build()

    private val _products = MutableStateFlow<List<ProductDetails>>(emptyList())
    val products = _products.asStateFlow()

    fun connect() {
        billingClient.startConnection(object : BillingClientStateListener {
            override fun onBillingSetupFinished(result: BillingResult) {
                if (result.responseCode == BillingClient.BillingResponseCode.OK) {
                    queryProducts()
                }
            }
            override fun onBillingServiceDisconnected() {
                // リトライ
            }
        })
    }

    private fun queryProducts() {
        val params = QueryProductDetailsParams.newBuilder()
            .setProductList(
                listOf(
                    QueryProductDetailsParams.Product.newBuilder()
                        .setProductId("premium_monthly")
                        .setProductType(BillingClient.ProductType.SUBS)
                        .build(),
                    QueryProductDetailsParams.Product.newBuilder()
                        .setProductId("remove_ads")
                        .setProductType(BillingClient.ProductType.INAPP)
                        .build()
                )
            ).build()

        billingClient.queryProductDetailsAsync(params) { result, productDetailsList ->
            if (result.responseCode == BillingClient.BillingResponseCode.OK) {
                _products.value = productDetailsList
            }
        }
    }

    fun purchase(productDetails: ProductDetails) {
        val flowParams = BillingFlowParams.newBuilder()
            .setProductDetailsParamsList(
                listOf(
                    BillingFlowParams.ProductDetailsParams.newBuilder()
                        .setProductDetails(productDetails)
                        .build()
                )
            ).build()
        billingClient.launchBillingFlow(activity, flowParams)
    }

    private fun handlePurchase(purchase: Purchase) {
        if (purchase.purchaseState == Purchase.PurchaseState.PURCHASED && !purchase.isAcknowledged) {
            val params = AcknowledgePurchaseParams.newBuilder()
                .setPurchaseToken(purchase.purchaseToken)
                .build()
            billingClient.acknowledgePurchase(params) {}
        }
    }
}
```

---

## Compose画面

```kotlin
@Composable
fun PurchaseScreen(billingManager: BillingManager) {
    val products by billingManager.products.collectAsStateWithLifecycle()

    LaunchedEffect(Unit) {
        billingManager.connect()
    }

    LazyColumn(Modifier.padding(16.dp)) {
        items(products) { product ->
            Card(
                Modifier
                    .fillMaxWidth()
                    .padding(vertical = 8.dp)
            ) {
                Column(Modifier.padding(16.dp)) {
                    Text(product.name, style = MaterialTheme.typography.titleMedium)
                    Text(product.description, style = MaterialTheme.typography.bodyMedium)
                    Spacer(Modifier.height(8.dp))

                    val price = product.subscriptionOfferDetails
                        ?.firstOrNull()?.pricingPhases?.pricingPhaseList
                        ?.firstOrNull()?.formattedPrice
                        ?: product.oneTimePurchaseOfferDetails?.formattedPrice

                    Button(onClick = { billingManager.purchase(product) }) {
                        Text("購入 $price")
                    }
                }
            }
        }
    }
}
```

---

## まとめ

- `BillingClient`でGoogle Playに接続
- `queryProductDetailsAsync`で商品情報取得
- `launchBillingFlow`で購入フロー開始
- `acknowledgePurchase`で購入を承認（必須）
- `INAPP`は消耗品/非消耗品、`SUBS`はサブスクリプション
- 購入検証はサーバーサイドで実施推奨

---

8種類のAndroidアプリテンプレート（課金対応設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Google Play公開](https://zenn.dev/myougatheaxo/articles/android-google-play-release-2026)
- [In-App Review](https://zenn.dev/myougatheaxo/articles/android-compose-in-app-review-2026)
- [アプリ収益化](https://zenn.dev/myougatheaxo/articles/android-app-monetization-2026)
