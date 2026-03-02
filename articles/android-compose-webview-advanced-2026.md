---
title: "WebView詳細ガイド — JS連携/Cookie/ダウンロード"
emoji: "🌐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "webview"]
published: true
---

## この記事で学べること

ComposeでのWebView詳細（JavaScript連携、Cookie管理、ファイルダウンロード、エラーハンドリング）を解説します。

---

## 基本のWebView (Accompanist)

```kotlin
// implementation("com.google.accompanist:accompanist-webview:0.34.0")

@Composable
fun WebViewScreen(url: String) {
    val state = rememberWebViewState(url)
    val navigator = rememberWebViewNavigator()

    Column {
        // ローディングバー
        if (state.isLoading) {
            LinearProgressIndicator(Modifier.fillMaxWidth())
        }

        WebView(
            state = state,
            navigator = navigator,
            modifier = Modifier.fillMaxSize(),
            onCreated = { webView ->
                webView.settings.apply {
                    javaScriptEnabled = true
                    domStorageEnabled = true
                    setSupportZoom(true)
                }
            }
        )
    }

    // 戻るボタン制御
    BackHandler(navigator.canGoBack) {
        navigator.navigateBack()
    }
}
```

---

## JavaScript連携

```kotlin
@Composable
fun JsBridgeWebView() {
    val state = rememberWebViewState("file:///android_asset/index.html")
    var receivedData by remember { mutableStateOf("") }

    WebView(
        state = state,
        modifier = Modifier.fillMaxSize(),
        onCreated = { webView ->
            webView.settings.javaScriptEnabled = true

            // Kotlin→JS呼び出し用インターフェース
            webView.addJavascriptInterface(
                object {
                    @JavascriptInterface
                    fun sendToNative(data: String) {
                        receivedData = data
                    }

                    @JavascriptInterface
                    fun getToken(): String {
                        return "user_token_123"
                    }
                },
                "AndroidBridge"
            )
        }
    )

    // Kotlin→JS実行
    LaunchedEffect(Unit) {
        delay(2000) // ページ読み込み待ち
        state.webView?.evaluateJavascript(
            "javascript:updateTitle('Hello from Kotlin')",
            null
        )
    }
}

// HTML側
// <script>
//   AndroidBridge.sendToNative("data from JS");
//   const token = AndroidBridge.getToken();
// </script>
```

---

## WebViewClient カスタマイズ

```kotlin
@Composable
fun CustomWebView(url: String) {
    val state = rememberWebViewState(url)

    WebView(
        state = state,
        modifier = Modifier.fillMaxSize(),
        client = remember {
            object : AccompanistWebViewClient() {
                override fun shouldOverrideUrlLoading(
                    view: WebView?,
                    request: WebResourceRequest?
                ): Boolean {
                    val requestUrl = request?.url?.toString() ?: return false

                    // 外部リンクはブラウザで開く
                    if (!requestUrl.startsWith("https://myapp.example.com")) {
                        val intent = Intent(Intent.ACTION_VIEW, Uri.parse(requestUrl))
                        view?.context?.startActivity(intent)
                        return true
                    }
                    return false
                }

                override fun onReceivedError(
                    view: WebView?,
                    request: WebResourceRequest?,
                    error: WebResourceError?
                ) {
                    super.onReceivedError(view, request, error)
                    // エラーハンドリング
                }
            }
        }
    )
}
```

---

## Cookie管理

```kotlin
fun setupCookies(url: String, token: String) {
    val cookieManager = CookieManager.getInstance()
    cookieManager.setAcceptCookie(true)

    // Cookie設定
    cookieManager.setCookie(url, "auth_token=$token; Path=/; Secure; HttpOnly")
    cookieManager.setCookie(url, "lang=ja; Path=/")

    cookieManager.flush()
}

// Cookie取得
fun getCookies(url: String): String? {
    return CookieManager.getInstance().getCookie(url)
}

// Cookie全削除
fun clearAllCookies() {
    CookieManager.getInstance().removeAllCookies(null)
    CookieManager.getInstance().flush()
}
```

---

## ファイルダウンロード

```kotlin
@Composable
fun DownloadWebView(url: String) {
    val context = LocalContext.current
    val state = rememberWebViewState(url)

    WebView(
        state = state,
        modifier = Modifier.fillMaxSize(),
        onCreated = { webView ->
            webView.settings.javaScriptEnabled = true

            webView.setDownloadListener { downloadUrl, _, contentDisposition, mimeType, _ ->
                val request = DownloadManager.Request(Uri.parse(downloadUrl)).apply {
                    setMimeType(mimeType)
                    val fileName = URLUtil.guessFileName(downloadUrl, contentDisposition, mimeType)
                    setTitle(fileName)
                    setDestinationInExternalPublicDir(
                        Environment.DIRECTORY_DOWNLOADS, fileName
                    )
                    setNotificationVisibility(
                        DownloadManager.Request.VISIBILITY_VISIBLE_NOTIFY_COMPLETED
                    )
                }

                val dm = context.getSystemService<DownloadManager>()
                dm?.enqueue(request)
            }
        }
    )
}
```

---

## まとめ

- `rememberWebViewState`/`rememberWebViewNavigator`で状態管理
- `addJavascriptInterface`でKotlin⇔JS双方向通信
- `evaluateJavascript`でKotlinからJS実行
- `shouldOverrideUrlLoading`で外部リンク制御
- `CookieManager`で認証Cookie管理
- `setDownloadListener`でファイルダウンロード対応
- `BackHandler`で戻るボタン制御

---

8種類のAndroidアプリテンプレート（WebView統合済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [WebView Bridge](https://zenn.dev/myougatheaxo/articles/android-compose-webview-bridge-2026)
- [DeepLink](https://zenn.dev/myougatheaxo/articles/android-compose-deeplink-handler-2026)
- [セキュリティ](https://zenn.dev/myougatheaxo/articles/android-compose-security-best-practices-2026)
