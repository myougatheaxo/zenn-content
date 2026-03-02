---
title: "WebView完全ガイド — Compose統合/JavaScript連携/Cookie/セキュリティ"
emoji: "🌐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "webview"]
published: true
---

## この記事で学べること

**WebView**（Compose統合、JavaScript連携、Cookie管理、ナビゲーション制御、セキュリティ）を解説します。

---

## Compose WebView

```kotlin
// Accompanist WebView (com.google.accompanist:accompanist-webview)
@Composable
fun WebViewScreen(url: String) {
    val state = rememberWebViewState(url)
    val navigator = rememberWebViewNavigator()

    Column(Modifier.fillMaxSize()) {
        // ローディングインジケーター
        if (state.isLoading) {
            LinearProgressIndicator(Modifier.fillMaxWidth())
        }

        WebView(
            state = state,
            modifier = Modifier.fillMaxSize(),
            navigator = navigator,
            onCreated = { webView ->
                webView.settings.apply {
                    javaScriptEnabled = true
                    domStorageEnabled = true
                    setSupportZoom(true)
                }
            }
        )
    }

    // 戻るボタン処理
    BackHandler(navigator.canGoBack) {
        navigator.navigateBack()
    }
}
```

---

## AndroidView方式

```kotlin
@Composable
fun CustomWebView(
    url: String,
    onPageFinished: (String) -> Unit = {}
) {
    var webView by remember { mutableStateOf<WebView?>(null) }

    AndroidView(
        factory = { context ->
            WebView(context).apply {
                settings.apply {
                    javaScriptEnabled = true
                    domStorageEnabled = true
                    cacheMode = WebSettings.LOAD_DEFAULT
                }

                webViewClient = object : WebViewClient() {
                    override fun onPageFinished(view: WebView?, url: String?) {
                        url?.let { onPageFinished(it) }
                    }

                    override fun shouldOverrideUrlLoading(
                        view: WebView?,
                        request: WebResourceRequest?
                    ): Boolean {
                        // 外部リンクはブラウザで開く
                        val uri = request?.url ?: return false
                        return if (uri.host != "example.com") {
                            context.startActivity(Intent(Intent.ACTION_VIEW, uri))
                            true
                        } else false
                    }
                }

                loadUrl(url)
                webView = this
            }
        },
        modifier = Modifier.fillMaxSize()
    )

    BackHandler {
        webView?.let { if (it.canGoBack()) it.goBack() }
    }
}
```

---

## JavaScript連携

```kotlin
@Composable
fun JsBridgeWebView() {
    AndroidView(
        factory = { context ->
            WebView(context).apply {
                settings.javaScriptEnabled = true

                // Android → JavaScript
                addJavascriptInterface(object {
                    @JavascriptInterface
                    fun showToast(message: String) {
                        Toast.makeText(context, message, Toast.LENGTH_SHORT).show()
                    }

                    @JavascriptInterface
                    fun sendData(json: String) {
                        // JSからデータ受信
                    }
                }, "Android")

                // JavaScript → Android: Android.showToast("Hello")
                // JavaScript → Android: Android.sendData('{"key":"value"}')

                loadUrl("https://example.com")
            }
        }
    )
}

// JavaScript側から呼び出し:
// <button onclick="Android.showToast('Hello from JS!')">Click</button>
```

---

## Cookie管理

```kotlin
object WebViewCookieManager {
    fun setCookie(url: String, cookie: String) {
        CookieManager.getInstance().apply {
            setAcceptCookie(true)
            setCookie(url, cookie)
            flush()
        }
    }

    fun getCookies(url: String): String? {
        return CookieManager.getInstance().getCookie(url)
    }

    fun clearAll() {
        CookieManager.getInstance().removeAllCookies(null)
    }
}
```

---

## まとめ

| 機能 | 実装 |
|------|------|
| Compose統合 | `AndroidView` + `WebView` |
| JS連携 | `@JavascriptInterface` |
| 戻る制御 | `BackHandler` + `canGoBack` |
| Cookie | `CookieManager` |

- `AndroidView`でWebViewをCompose内に配置
- `@JavascriptInterface`でKotlin↔JavaScript通信
- `BackHandler`でWebView内の戻る操作を制御
- セキュリティ: 信頼できるURLのみ`javaScriptEnabled`

---

8種類のAndroidアプリテンプレート（WebView統合対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Retrofit/OkHttp](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-okhttp-2026)
- [ディープリンク](https://zenn.dev/myougatheaxo/articles/android-compose-deeplink-2026)
- [セキュリティ](https://zenn.dev/myougatheaxo/articles/android-compose-certificate-pinning-2026)
