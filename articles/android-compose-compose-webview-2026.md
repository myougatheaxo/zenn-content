---
title: "Compose WebView完全ガイド — AndroidView/JavaScript連携/カスタムWebView"
emoji: "🌐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "webview"]
published: true
---

## この記事で学べること

**Compose WebView**（AndroidView、WebView設定、JavaScript連携、カスタムWebViewClient）を解説します。

---

## 基本WebView

```kotlin
@SuppressLint("SetJavaScriptEnabled")
@Composable
fun WebViewScreen(url: String) {
    var isLoading by remember { mutableStateOf(true) }

    Box(Modifier.fillMaxSize()) {
        AndroidView(
            factory = { context ->
                WebView(context).apply {
                    settings.javaScriptEnabled = true
                    settings.domStorageEnabled = true
                    webViewClient = object : WebViewClient() {
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
            CircularProgressIndicator(Modifier.align(Alignment.Center))
        }
    }
}
```

---

## JavaScript連携

```kotlin
@SuppressLint("SetJavaScriptEnabled")
@Composable
fun JsBridgeWebView() {
    var message by remember { mutableStateOf("") }
    var webView by remember { mutableStateOf<WebView?>(null) }

    Column(Modifier.fillMaxSize()) {
        if (message.isNotEmpty()) {
            Text("JSからのメッセージ: $message", Modifier.padding(16.dp),
                style = MaterialTheme.typography.bodyMedium)
        }

        AndroidView(
            factory = { context ->
                WebView(context).apply {
                    settings.javaScriptEnabled = true
                    addJavascriptInterface(object {
                        @JavascriptInterface
                        fun sendMessage(msg: String) {
                            message = msg
                        }
                    }, "Android")
                    webView = this
                    loadData("""
                        <html><body>
                        <button onclick="Android.sendMessage('Hello from JS!')">
                            JSからメッセージ送信
                        </button>
                        </body></html>
                    """.trimIndent(), "text/html", "utf-8")
                }
            },
            modifier = Modifier.weight(1f)
        )

        Button(onClick = {
            webView?.evaluateJavascript("document.title") { title ->
                message = "タイトル: $title"
            }
        }, Modifier.padding(16.dp)) { Text("JSを実行") }
    }
}
```

---

## 戻る対応

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
                settings.javaScriptEnabled = true
                webViewClient = WebViewClient()
                webView = this
                loadUrl(url)
            }
        },
        modifier = Modifier.fillMaxSize()
    )
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `AndroidView` | WebViewの配置 |
| `WebViewClient` | ページ制御 |
| `addJavascriptInterface` | JS→Android通信 |
| `evaluateJavascript` | Android→JS実行 |

- `AndroidView`でComposeにWebViewを埋め込み
- `@JavascriptInterface`でJSからAndroidのメソッドを呼び出し
- `evaluateJavascript`でAndroidからJSを実行
- `BackHandler`でWebView内のナビゲーション戻るに対応

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Interop](https://zenn.dev/myougatheaxo/articles/android-compose-compose-interop-2026)
- [Compose AppLink](https://zenn.dev/myougatheaxo/articles/android-compose-compose-app-link-2026)
- [Compose DeepLink](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-deep-link-2026)
