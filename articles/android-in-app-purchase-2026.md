---
title: "アプリ内課金（In-App Purchase）実装ガイド — Google Play Billing"
emoji: "💰"
type: "tech"
topics: ["android", "kotlin", "billing", "googleplay"]
published: true
---

## この記事で学べること

**Google Play Billing Library**を使ったアプリ内課金（一回購入・サブスクリプション）の実装方法を解説します。

---

## セットアップ

```kotlin
// build.gradle.kts
dependencies {
    implementation("com.android.billingclient:billing-ktx:7.0.0")
}
```

---

## BillingClient初期化

```kotlin
class BillingManager(private val context: Context) {
    private var billingClient: BillingClient? = null

    fun initialize() {
        billingClient = BillingClient.newBuilder(context)
            .setListener { billingResult, purchases ->
                if (billingResult.responseCode == BillingClient.BillingResponseCode.OK) {
                    purchases?.forEach { purchase ->
                        handlePurchase(purchase)
                    }
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

            override fun onBillingServiceDisconnected() {
                // 再接続ロジック
            }
        })
    }
}
```

---

## 商品情報の取得

```kotlin
private suspend fun queryProducts() {
    val productList = listOf(
        QueryProductDetailsParams.Product.newBuilder()
            .setProductId("premium_upgrade")
            .setProductType(BillingClient.ProductType.INAPP)
            .build()
    )

    val params = QueryProductDetailsParams.newBuilder()
        .setProductList(productList)
        .build()

    val result = billingClient?.queryProductDetails(params)
    result?.productDetailsList?.forEach { productDetails ->
        // 商品情報を表示
        val price = productDetails.oneTimePurchaseOfferDetails?.formattedPrice
    }
}
```

---

## 購入フローの開始

```kotlin
fun launchPurchase(activity: Activity, productDetails: ProductDetails) {
    val productDetailsParams = BillingFlowParams.ProductDetailsParams.newBuilder()
        .setProductDetails(productDetails)
        .build()

    val billingFlowParams = BillingFlowParams.newBuilder()
        .setProductDetailsParamsList(listOf(productDetailsParams))
        .build()

    billingClient?.launchBillingFlow(activity, billingFlowParams)
}
```

---

## 購入の確認（Acknowledge）

```kotlin
private suspend fun handlePurchase(purchase: Purchase) {
    if (purchase.purchaseState == Purchase.PurchaseState.PURCHASED) {
        if (!purchase.isAcknowledged) {
            val params = AcknowledgePurchaseParams.newBuilder()
                .setPurchaseToken(purchase.purchaseToken)
                .build()

            billingClient?.acknowledgePurchase(params)
        }
        // 機能をアンロック
        unlockPremium()
    }
}
```

**重要**: 3日以内にacknowledgeしないと自動返金される。

---

## サブスクリプション

```kotlin
// サブスクリプション商品の取得
val subProduct = QueryProductDetailsParams.Product.newBuilder()
    .setProductId("monthly_premium")
    .setProductType(BillingClient.ProductType.SUBS)
    .build()

// サブスクリプション購入
fun launchSubscription(
    activity: Activity,
    productDetails: ProductDetails,
    offerToken: String
) {
    val params = BillingFlowParams.ProductDetailsParams.newBuilder()
        .setProductDetails(productDetails)
        .setOfferToken(offerToken)
        .build()

    val flowParams = BillingFlowParams.newBuilder()
        .setProductDetailsParamsList(listOf(params))
        .build()

    billingClient?.launchBillingFlow(activity, flowParams)
}
```

---

## 購入履歴の復元

```kotlin
suspend fun restorePurchases() {
    val params = QueryPurchasesParams.newBuilder()
        .setProductType(BillingClient.ProductType.INAPP)
        .build()

    val result = billingClient?.queryPurchasesAsync(params)
    result?.purchasesList?.forEach { purchase ->
        if (purchase.purchaseState == Purchase.PurchaseState.PURCHASED) {
            unlockPremium()
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
    val isPremium by billingManager.isPremium.collectAsStateWithLifecycle()
    val activity = LocalContext.current as Activity

    Column(Modifier.padding(16.dp)) {
        if (isPremium) {
            Card(
                colors = CardDefaults.cardColors(
                    containerColor = MaterialTheme.colorScheme.primaryContainer
                )
            ) {
                Text("プレミアム会員です", Modifier.padding(16.dp))
            }
        } else {
            products.forEach { product ->
                Card(Modifier.fillMaxWidth().padding(vertical = 4.dp)) {
                    Column(Modifier.padding(16.dp)) {
                        Text(product.name, style = MaterialTheme.typography.titleMedium)
                        Text(product.description)
                        Spacer(Modifier.height(8.dp))
                        Button(onClick = {
                            billingManager.launchPurchase(activity, product)
                        }) {
                            Text("${product.formattedPrice}で購入")
                        }
                    }
                }
            }
        }
    }
}
```

---

## まとめ

- `BillingClient`で初期化・接続
- `queryProductDetails`で商品情報取得
- `launchBillingFlow`で購入UI表示
- `acknowledgePurchase`で購入確認（3日以内必須）
- `queryPurchasesAsync`で購入復元
- サブスクは`ProductType.SUBS` + `offerToken`

---

8種類のAndroidアプリテンプレート（課金機能追加可能な設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Google Play公開ガイド](https://zenn.dev/myougatheaxo/articles/android-google-play-publish-2026)
- [セキュリティチェックリスト](https://zenn.dev/myougatheaxo/articles/android-security-checklist-2026)
- [Firebase Authentication実装](https://zenn.dev/myougatheaxo/articles/android-firebase-auth-2026)
