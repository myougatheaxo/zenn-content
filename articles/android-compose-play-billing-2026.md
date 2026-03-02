---
title: "Google Play課金完全ガイド — BillingClient/サブスクリプション/消耗品/確認"
emoji: "💳"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "billing"]
published: true
---

## この記事で学べること

**Google Play課金**（BillingClient、サブスクリプション、消耗品購入、購入確認）を解説します。

---

## BillingClient

```kotlin
class BillingRepository @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val _purchases = MutableStateFlow<List<Purchase>>(emptyList())
    val purchases: StateFlow<List<Purchase>> = _purchases.asStateFlow()

    private val billingClient = BillingClient.newBuilder(context)
        .setListener { billingResult, purchases ->
            if (billingResult.responseCode == BillingClient.BillingResponseCode.OK && purchases != null) {
                _purchases.value = purchases
                purchases.forEach { purchase -> acknowledgePurchase(purchase) }
            }
        }
        .enablePendingPurchases()
        .build()

    fun connect() {
        billingClient.startConnection(object : BillingClientStateListener {
            override fun onBillingSetupFinished(result: BillingResult) {
                if (result.responseCode == BillingClient.BillingResponseCode.OK) {
                    queryProducts()
                }
            }
            override fun onBillingServiceDisconnected() { connect() }
        })
    }

    private fun queryProducts() {
        val params = QueryProductDetailsParams.newBuilder()
            .setProductList(listOf(
                QueryProductDetailsParams.Product.newBuilder()
                    .setProductId("premium_monthly")
                    .setProductType(BillingClient.ProductType.SUBS)
                    .build()
            ))
            .build()

        billingClient.queryProductDetailsAsync(params) { _, productDetails ->
            // 商品情報を保持
        }
    }

    fun launchPurchase(activity: Activity, productDetails: ProductDetails) {
        val offerToken = productDetails.subscriptionOfferDetails?.firstOrNull()?.offerToken ?: return
        val params = BillingFlowParams.newBuilder()
            .setProductDetailsParamsList(listOf(
                BillingFlowParams.ProductDetailsParams.newBuilder()
                    .setProductDetails(productDetails)
                    .setOfferToken(offerToken)
                    .build()
            ))
            .build()

        billingClient.launchBillingFlow(activity, params)
    }

    private fun acknowledgePurchase(purchase: Purchase) {
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

## Compose UI

```kotlin
@Composable
fun PremiumScreen(viewModel: BillingViewModel = hiltViewModel()) {
    val isPremium by viewModel.isPremium.collectAsStateWithLifecycle(false)
    val activity = LocalContext.current as Activity

    Column(
        Modifier.fillMaxSize().padding(32.dp),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        if (isPremium) {
            Icon(Icons.Default.Star, null, tint = Color(0xFFFFD700), modifier = Modifier.size(64.dp))
            Text("プレミアム会員", style = MaterialTheme.typography.headlineMedium)
        } else {
            Text("プレミアムにアップグレード", style = MaterialTheme.typography.headlineSmall)
            Spacer(Modifier.height(16.dp))

            listOf("広告非表示", "全機能解放", "優先サポート").forEach { feature ->
                Row(Modifier.padding(vertical = 4.dp)) {
                    Icon(Icons.Default.Check, null, tint = Color(0xFF4CAF50))
                    Spacer(Modifier.width(8.dp))
                    Text(feature)
                }
            }

            Spacer(Modifier.height(24.dp))
            Button(onClick = { viewModel.purchase(activity) }, Modifier.fillMaxWidth()) {
                Text("月額 ¥480 で購入")
            }
        }
    }
}
```

---

## まとめ

| 種別 | ProductType |
|------|------------|
| サブスク | `SUBS` |
| 消耗品 | `INAPP` |

- `BillingClient`でGoogle Play課金フロー
- `acknowledgePurchase`で購入確認（3日以内必須）
- サブスクリプションは`offerToken`で起動
- `queryPurchasesAsync`で購入状態を復元

---

8種類のAndroidアプリテンプレート（課金機能対応可）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [アプリ内レビュー](https://zenn.dev/myougatheaxo/articles/android-compose-in-app-review-2026)
- [DataStore](https://zenn.dev/myougatheaxo/articles/android-compose-datastore-preferences-2026)
- [Google Play公開](https://zenn.dev/myougatheaxo/articles/android-compose-google-play-release-2026)
