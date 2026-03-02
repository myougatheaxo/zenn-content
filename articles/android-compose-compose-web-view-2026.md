---
title: "WebView in Compose完全ガイド — AndroidView/JavaScript連携/ナビゲーション"
emoji: "🌐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "webview"]
published: true
---

## この記事で学べること

**WebView in Compose**（AndroidView、JavaScript連携、WebViewClient、ナビゲーション制御）を解説します。

---

## WebView基本

```kotlin
@Composable
fun BasicWebView(url: String) {
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
            CircularProgressIndicator(Modifier.align(Alignment.Center))
        }
    }
}
```

---

## ナビゲーション制御

```kotlin
@Composable
fun WebViewWithNavigation(initialUrl: String) {
    var webView by remember { mutableStateOf<WebView?>(null) }
    var canGoBack by remember { mutableStateOf(false) }
    var currentUrl by remember { mutableStateOf(initialUrl) }

    Column(Modifier.fillMaxSize()) {
        // ツールバー
        TopAppBar(
            title = { Text(currentUrl, maxLines = 1, overflow = TextOverflow.Ellipsis) },
            navigationIcon = {
                IconButton(onClick = { webView?.goBack() }, enabled = canGoBack) {
                    Icon(Icons.AutoMirrored.Filled.ArrowBack, "戻る")
                }
            },
            actions = {
                IconButton(onClick = { webView?.reload() }) {
                    Icon(Icons.Default.Refresh, "リロード")
                }
            }
        )

        AndroidView(
            factory = { context ->
                WebView(context).apply {
                    webViewClient = object : WebViewClient() {
                        override fun doUpdateVisitedHistory(view: WebView?, url: String?, isReload: Boolean) {
                            canGoBack = view?.canGoBack() ?: false
                            currentUrl = url ?: ""
                        }
                    }
                    settings.javaScriptEnabled = true
                    loadUrl(initialUrl)
                    webView = this
                }
            },
            modifier = Modifier.weight(1f)
        )
    }

    BackHandler(canGoBack) { webView?.goBack() }
}
```

---

## JavaScript連携

```kotlin
@Composable
fun WebViewWithJsBridge() {
    var message by remember { mutableStateOf("") }

    Column(Modifier.fillMaxSize()) {
        if (message.isNotEmpty()) {
            Card(Modifier.fillMaxWidth().padding(8.dp)) {
                Text("JS: $message", Modifier.padding(12.dp))
            }
        }

        AndroidView(
            factory = { context ->
                WebView(context).apply {
                    settings.javaScriptEnabled = true
                    addJavascriptInterface(object {
                        @JavascriptInterface
                        fun sendMessage(msg: String) { message = msg }
                    }, "Android")
                    loadData("""
                        <html><body>
                        <button onclick="Android.sendMessage('ボタンが押されました')">
                            メッセージを送信
                        </button>
                        </body></html>
                    """.trimIndent(), "text/html", "utf-8")
                }
            },
            modifier = Modifier.weight(1f)
        )
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `AndroidView` | WebViewをCompose内に配置 |
| `WebViewClient` | ページ読み込み制御 |
| `JavascriptInterface` | JS↔Kotlin通信 |
| `BackHandler` | 戻るボタン制御 |

- `AndroidView`でWebViewをCompose内に埋め込み
- `WebViewClient`でナビゲーション・読み込み管理
- `JavascriptInterface`でJS-Kotlin間通信
- `BackHandler`でWebView内の戻る制御

---

8種類のAndroidアプリテンプレート（WebView対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [BackHandler](https://zenn.dev/myougatheaxo/articles/android-compose-compose-back-handler-2026)
- [ProgressIndicator](https://zenn.dev/myougatheaxo/articles/android-compose-compose-progress-indicator-2026)
- [DeepLink](https://zenn.dev/myougatheaxo/articles/android-compose-compose-deep-link-2026)
