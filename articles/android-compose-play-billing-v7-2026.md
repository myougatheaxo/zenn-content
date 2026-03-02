---
title: "Play Billing Library v7完全ガイド — サブスクリプション/ワンタイム購入"
emoji: "💳"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "billing"]
published: true
---

## この記事で学べること

**Play Billing Library v7**（サブスクリプション、ワンタイム購入、購入検証、UI統合、テスト）を解説します。

---

## セットアップ

```kotlin
dependencies {
    implementation("com.android.billingclient:billing-ktx:7.1.1")
}
```

---

## BillingManager

```kotlin
class BillingManager @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val _purchases = MutableStateFlow<List<Purchase>>(emptyList())
    val purchases: StateFlow<List<Purchase>> = _purchases

    private val billingClient = BillingClient.newBuilder(context)
        .setListener { billingResult, purchases ->
            if (billingResult.responseCode == BillingClient.BillingResponseCode.OK && purchases != null) {
                _purchases.value = purchases
                purchases.forEach { handlePurchase(it) }
            }
        }
        .enablePendingPurchases(PendingPurchasesParams.newBuilder().enableOneTimeProducts().build())
        .build()

    fun connect(onReady: () -> Unit) {
        billingClient.startConnection(object : BillingClientStateListener {
            override fun onBillingSetupFinished(result: BillingResult) {
                if (result.responseCode == BillingClient.BillingResponseCode.OK) onReady()
            }
            override fun onBillingServiceDisconnected() { /* 再接続 */ }
        })
    }

    suspend fun queryProducts(productIds: List<String>, type: String): List<ProductDetails> {
        val params = QueryProductDetailsParams.newBuilder().setProductList(
            productIds.map {
                QueryProductDetailsParams.Product.newBuilder()
                    .setProductId(it)
                    .setProductType(type)
                    .build()
            }
        ).build()
        val (result, details) = billingClient.queryProductDetails(params)
        return if (result.responseCode == BillingClient.BillingResponseCode.OK) details.orEmpty() else emptyList()
    }

    fun launchPurchaseFlow(activity: Activity, productDetails: ProductDetails, offerToken: String? = null) {
        val builder = BillingFlowParams.ProductDetailsParams.newBuilder()
            .setProductDetails(productDetails)
        offerToken?.let { builder.setOfferToken(it) }

        val flowParams = BillingFlowParams.newBuilder()
            .setProductDetailsParamsList(listOf(builder.build()))
            .build()
        billingClient.launchBillingFlow(activity, flowParams)
    }

    private fun handlePurchase(purchase: Purchase) {
        if (purchase.purchaseState == Purchase.PurchaseState.PURCHASED && !purchase.isAcknowledged) {
            val params = AcknowledgePurchaseParams.newBuilder()
                .setPurchaseToken(purchase.purchaseToken)
                .build()
            billingClient.acknowledgePurchase(params) { /* 確認完了 */ }
        }
    }
}
```

---

## Compose画面

```kotlin
@Composable
fun PurchaseScreen(billingManager: BillingManager) {
    val activity = LocalContext.current as Activity
    var products by remember { mutableStateOf<List<ProductDetails>>(emptyList()) }

    LaunchedEffect(Unit) {
        billingManager.connect {
            products = billingManager.queryProducts(
                listOf("premium_monthly", "premium_yearly"),
                BillingClient.ProductType.SUBS
            )
        }
    }

    Column(Modifier.padding(16.dp)) {
        Text("プレミアムプラン", style = MaterialTheme.typography.headlineMedium)
        Spacer(Modifier.height(16.dp))

        products.forEach { product ->
            Card(Modifier.fillMaxWidth().padding(vertical = 4.dp)) {
                Row(
                    Modifier.fillMaxWidth().clickable {
                        val offer = product.subscriptionOfferDetails?.firstOrNull()
                        billingManager.launchPurchaseFlow(activity, product, offer?.offerToken)
                    }.padding(16.dp),
                    horizontalArrangement = Arrangement.SpaceBetween
                ) {
                    Column {
                        Text(product.name, style = MaterialTheme.typography.titleMedium)
                        Text(product.description, style = MaterialTheme.typography.bodySmall)
                    }
                    product.subscriptionOfferDetails?.firstOrNull()?.let { offer ->
                        Text(
                            offer.pricingPhases.pricingPhaseList.first().formattedPrice,
                            style = MaterialTheme.typography.titleLarge,
                            color = MaterialTheme.colorScheme.primary
                        )
                    }
                }
            }
        }
    }
}
```

---

## まとめ

| 機能 | 実装 |
|------|------|
| 接続 | `BillingClient.startConnection` |
| 商品取得 | `queryProductDetails()` |
| 購入開始 | `launchBillingFlow()` |
| 確認 | `acknowledgePurchase()` |

- Billing Library v7でKotlinコルーチン対応
- `PendingPurchases`で保留中購入を処理
- サーバーサイド検証でセキュリティ強化
- テスト用アカウントでサンドボックス購入

---

8種類のAndroidアプリテンプレート（課金機能対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [アプリ内購入](https://zenn.dev/myougatheaxo/articles/android-compose-in-app-purchase-2026)
- [AdMob](https://zenn.dev/myougatheaxo/articles/android-compose-admob-integration-2026)
- [セキュリティ](https://zenn.dev/myougatheaxo/articles/android-compose-security-best-practices-2026)
