---
title: "WebView + Compose — Webコンテンツ表示とJavaScript連携"
emoji: "🌐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "webview"]
published: true
---

## この記事で学べること

ComposeアプリにWebViewを統合し、Web表示・JavaScript連携・ナビゲーション制御を実装する方法を解説します。

---

## 基本のWebView

```kotlin
@Composable
fun BasicWebView(url: String) {
    AndroidView(
        factory = { context ->
            WebView(context).apply {
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

## ナビゲーション制御

```kotlin
@Composable
fun WebViewWithNavigation(url: String) {
    var webView by remember { mutableStateOf<WebView?>(null) }
    var canGoBack by remember { mutableStateOf(false) }
    var currentUrl by remember { mutableStateOf(url) }
    var isLoading by remember { mutableStateOf(true) }

    Column(Modifier.fillMaxSize()) {
        // ツールバー
        TopAppBar(
            title = { Text(currentUrl, maxLines = 1, overflow = TextOverflow.Ellipsis) },
            navigationIcon = {
                IconButton(
                    onClick = { webView?.goBack() },
                    enabled = canGoBack
                ) {
                    Icon(Icons.AutoMirrored.Filled.ArrowBack, "戻る")
                }
            },
            actions = {
                IconButton(onClick = { webView?.reload() }) {
                    Icon(Icons.Default.Refresh, "更新")
                }
            }
        )

        if (isLoading) {
            LinearProgressIndicator(Modifier.fillMaxWidth())
        }

        AndroidView(
            factory = { context ->
                WebView(context).apply {
                    webViewClient = object : WebViewClient() {
                        override fun onPageStarted(view: WebView?, url: String?, favicon: android.graphics.Bitmap?) {
                            isLoading = true
                            currentUrl = url ?: ""
                        }
                        override fun onPageFinished(view: WebView?, url: String?) {
                            isLoading = false
                            canGoBack = view?.canGoBack() == true
                        }
                    }
                    settings.javaScriptEnabled = true
                    loadUrl(url)
                    webView = this
                }
            },
            modifier = Modifier.fillMaxSize()
        )
    }

    BackHandler(enabled = canGoBack) {
        webView?.goBack()
    }
}
```

---

## JavaScript連携

```kotlin
@Composable
fun WebViewWithJsBridge() {
    val context = LocalContext.current

    AndroidView(
        factory = {
            WebView(context).apply {
                settings.javaScriptEnabled = true

                // KotlinからJavaScriptを呼ぶ
                addJavascriptInterface(object {
                    @JavascriptInterface
                    fun showToast(message: String) {
                        Toast.makeText(context, message, Toast.LENGTH_SHORT).show()
                    }

                    @JavascriptInterface
                    fun sendData(json: String) {
                        // JSON解析してアプリ内で処理
                    }
                }, "AndroidBridge")

                loadUrl("file:///android_asset/index.html")
            }
        },
        modifier = Modifier.fillMaxSize()
    )
}
```

HTML側:
```html
<button onclick="AndroidBridge.showToast('Hello from Web!')">
    Androidに通知
</button>
```

---

## HTMLを直接表示

```kotlin
@Composable
fun HtmlContentView(htmlContent: String) {
    AndroidView(
        factory = { context ->
            WebView(context).apply {
                webViewClient = WebViewClient()
                settings.javaScriptEnabled = false
                loadDataWithBaseURL(
                    null,
                    htmlContent,
                    "text/html",
                    "UTF-8",
                    null
                )
            }
        },
        modifier = Modifier.fillMaxSize()
    )
}
```

---

## セキュリティ設定

```kotlin
WebView(context).apply {
    settings.apply {
        javaScriptEnabled = true
        domStorageEnabled = true
        // セキュリティ
        allowFileAccess = false
        allowContentAccess = false
        setSupportMultipleWindows(false)
        javaScriptCanOpenWindowsAutomatically = false
    }

    webViewClient = object : WebViewClient() {
        override fun shouldOverrideUrlLoading(
            view: WebView?,
            request: WebResourceRequest?
        ): Boolean {
            val host = request?.url?.host ?: return false
            // 許可するドメインのみ
            return !host.endsWith("example.com")
        }
    }
}
```

---

## まとめ

- `AndroidView` + `WebView`でCompose統合
- `WebViewClient`でページ遷移・読み込み制御
- `addJavascriptInterface`でKotlin↔JavaScript連携
- `BackHandler`で戻るボタン対応
- セキュリティ: `allowFileAccess = false`、ドメイン制限

---

8種類のAndroidアプリテンプレート（WebView統合可能な設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Navigation完全ガイド](https://zenn.dev/myougatheaxo/articles/android-navigation-compose-2026)
- [Deep Link実装ガイド](https://zenn.dev/myougatheaxo/articles/android-deeplink-guide-2026)
- [セキュリティチェックリスト](https://zenn.dev/myougatheaxo/articles/android-security-checklist-2026)
