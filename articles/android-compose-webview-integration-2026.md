---
title: "WebView + Compose完全ガイド — HTML表示/JS連携/Cookie管理"
emoji: "🌍"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "webview"]
published: true
---

## この記事で学べること

**WebView + Compose**（HTML表示、JavaScript連携、Cookie管理、ナビゲーション制御）を解説します。

---

## 基本WebView

```kotlin
@Composable
fun WebViewScreen(url: String) {
    var isLoading by remember { mutableStateOf(true) }

    Box(Modifier.fillMaxSize()) {
        AndroidView(
            factory = { context ->
                WebView(context).apply {
                    webViewClient = object : WebViewClient() {
                        override fun onPageFinished(view: WebView?, url: String?) {
                            isLoading = false
                        }
                    }
                    settings.javaScriptEnabled = true
                    settings.domStorageEnabled = true
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

## JavaScript連携

```kotlin
@Composable
fun WebViewWithJsBridge() {
    var webView by remember { mutableStateOf<WebView?>(null) }
    var messageFromJs by remember { mutableStateOf("") }

    Column(Modifier.fillMaxSize()) {
        AndroidView(
            factory = { context ->
                WebView(context).apply {
                    webView = this
                    settings.javaScriptEnabled = true

                    addJavascriptInterface(object {
                        @JavascriptInterface
                        fun postMessage(message: String) {
                            messageFromJs = message
                        }
                    }, "Android")

                    loadData("""
                        <html><body>
                        <button onclick="Android.postMessage('Hello from JS!')">
                            Send to Android
                        </button>
                        </body></html>
                    """.trimIndent(), "text/html", "UTF-8")
                }
            },
            modifier = Modifier.weight(1f)
        )

        if (messageFromJs.isNotEmpty()) {
            Text("JS: $messageFromJs", Modifier.padding(16.dp))
        }

        Button(onClick = {
            webView?.evaluateJavascript("document.title") { title ->
                messageFromJs = "Title: $title"
            }
        }) { Text("JSを実行") }
    }
}
```

---

## 戻るボタン対応

```kotlin
@Composable
fun WebViewWithBackHandler(url: String) {
    var webView by remember { mutableStateOf<WebView?>(null) }

    BackHandler(enabled = webView?.canGoBack() == true) {
        webView?.goBack()
    }

    AndroidView(
        factory = { context ->
            WebView(context).apply {
                webView = this
                webViewClient = WebViewClient()
                settings.javaScriptEnabled = true
                loadUrl(url)
            }
        },
        modifier = Modifier.fillMaxSize()
    )
}
```

---

## まとめ

| 機能 | 実装 |
|------|------|
| HTML表示 | `loadUrl()` / `loadData()` |
| JS実行 | `evaluateJavascript()` |
| JS→Android | `@JavascriptInterface` |
| 戻る | `BackHandler` + `canGoBack()` |
| Cookie | `CookieManager` |

- `AndroidView`でWebViewをComposeに統合
- `@JavascriptInterface`でJS→Kotlin通信
- `BackHandler`でWebView内ナビゲーション

---

8種類のAndroidアプリテンプレート（WebView対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose⇔View相互運用](https://zenn.dev/myougatheaxo/articles/android-compose-compose-interop-2026)
- [Deep Link](https://zenn.dev/myougatheaxo/articles/android-compose-app-links-deep-link-2026)
- [Share/Intent](https://zenn.dev/myougatheaxo/articles/android-compose-share-intent-2026)
