---
title: "WebView + Compose連携ガイド — JavaScript Bridge/Cookie管理"
emoji: "🌐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "webview"]
published: true
---

## この記事で学べること

Composeでの**WebView連携**（JavaScript Bridge、Cookie管理、ナビゲーション）を解説します。

---

## 基本のWebView

```kotlin
@Composable
fun BasicWebView(url: String) {
    var isLoading by remember { mutableStateOf(true) }

    Box(Modifier.fillMaxSize()) {
        AndroidView(
            factory = { context ->
                WebView(context).apply {
                    settings.javaScriptEnabled = true
                    settings.domStorageEnabled = true
                    webViewClient = object : WebViewClient() {
                        override fun onPageStarted(view: WebView?, url: String?, favicon: Bitmap?) {
                            isLoading = true
                        }
                        override fun onPageFinished(view: WebView?, url: String?) {
                            isLoading = false
                        }
                    }
                    loadUrl(url)
                }
            },
            modifier = Modifier.fillMaxSize()
        )

        if (isLoading) {
            LinearProgressIndicator(Modifier.fillMaxWidth().align(Alignment.TopCenter))
        }
    }
}
```

---

## JavaScript Bridge

```kotlin
class WebAppInterface(private val onMessage: (String) -> Unit) {
    @JavascriptInterface
    fun postMessage(message: String) {
        onMessage(message)
    }
}

@Composable
fun WebViewWithBridge() {
    var receivedMessage by remember { mutableStateOf("") }
    var webView by remember { mutableStateOf<WebView?>(null) }

    Column(Modifier.fillMaxSize()) {
        AndroidView(
            factory = { context ->
                WebView(context).apply {
                    settings.javaScriptEnabled = true
                    addJavascriptInterface(
                        WebAppInterface { msg -> receivedMessage = msg },
                        "Android"
                    )
                    loadUrl("file:///android_asset/index.html")
                    webView = this
                }
            },
            modifier = Modifier.weight(1f)
        )

        // Kotlinからjsを実行
        Button(onClick = {
            webView?.evaluateJavascript("updateFromNative('Hello from Kotlin')") {}
        }) {
            Text("JSに送信")
        }

        if (receivedMessage.isNotEmpty()) {
            Text("JSから受信: $receivedMessage")
        }
    }
}
```

---

## 戻る制御

```kotlin
@Composable
fun WebViewWithBackHandler(url: String) {
    var webView by remember { mutableStateOf<WebView?>(null) }
    var canGoBack by remember { mutableStateOf(false) }

    BackHandler(enabled = canGoBack) {
        webView?.goBack()
    }

    AndroidView(
        factory = { context ->
            WebView(context).apply {
                settings.javaScriptEnabled = true
                webViewClient = object : WebViewClient() {
                    override fun doUpdateVisitedHistory(view: WebView?, url: String?, isReload: Boolean) {
                        canGoBack = view?.canGoBack() == true
                    }
                }
                loadUrl(url)
                webView = this
            }
        }
    )
}
```

---

## まとめ

- `AndroidView { WebView }`でCompose内にWebView
- `WebViewClient`でページ読み込みイベント処理
- `@JavascriptInterface`でKotlin↔JavaScript通信
- `evaluateJavascript()`でKotlinからJS実行
- `BackHandler`でWebView内のページ戻り制御
- `settings.javaScriptEnabled`は必要な箇所のみ有効化

---

8種類のAndroidアプリテンプレート（WebView対応可能）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [WebView Composeガイド](https://zenn.dev/myougatheaxo/articles/android-webview-compose-2026)
- [View Interopガイド](https://zenn.dev/myougatheaxo/articles/compose-view-interop-2026)
- [セキュリティチェックリスト](https://zenn.dev/myougatheaxo/articles/android-security-checklist-2026)
