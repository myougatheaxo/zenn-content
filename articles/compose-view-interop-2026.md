---
title: "Compose ↔ View相互運用ガイド — AndroidViewとComposeView"
emoji: "🔀"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "interop"]
published: true
---

## この記事で学べること

Composeと従来のAndroid Viewを**相互に組み込む方法**を解説します。

---

## AndroidView: Compose内にViewを配置

```kotlin
@Composable
fun LegacyMapView() {
    AndroidView(
        factory = { context ->
            // View の作成（1回だけ実行）
            MapView(context).apply {
                onCreate(Bundle())
            }
        },
        update = { mapView ->
            // Recomposeのたびに実行
            mapView.getMapAsync { map ->
                map.moveCamera(CameraUpdateFactory.newLatLng(LatLng(35.68, 139.76)))
            }
        },
        modifier = Modifier
            .fillMaxWidth()
            .height(300.dp)
    )
}
```

---

## AndroidView: WebView

```kotlin
@Composable
fun WebContent(url: String) {
    AndroidView(
        factory = { context ->
            WebView(context).apply {
                settings.javaScriptEnabled = true
                webViewClient = WebViewClient()
            }
        },
        update = { webView ->
            webView.loadUrl(url)
        },
        modifier = Modifier.fillMaxSize()
    )
}
```

---

## AndroidView: AdMob広告

```kotlin
@Composable
fun BannerAd() {
    AndroidView(
        factory = { context ->
            AdView(context).apply {
                setAdSize(AdSize.BANNER)
                adUnitId = "ca-app-pub-xxxxx/yyyyy"
                loadAd(AdRequest.Builder().build())
            }
        },
        modifier = Modifier.fillMaxWidth()
    )
}
```

---

## ComposeView: View内にComposeを配置

```kotlin
// Fragment内でCompose使用
class SettingsFragment : Fragment() {
    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        return ComposeView(requireContext()).apply {
            setViewCompositionStrategy(
                ViewCompositionStrategy.DisposeOnViewTreeLifecycleDestroyed
            )
            setContent {
                MaterialTheme {
                    SettingsScreen()
                }
            }
        }
    }
}
```

---

## XML内でComposeView

```xml
<!-- res/layout/activity_main.xml -->
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <TextView
        android:id="@+id/title"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="レガシーView" />

    <androidx.compose.ui.platform.ComposeView
        android:id="@+id/compose_view"
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />

</LinearLayout>
```

```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        findViewById<ComposeView>(R.id.compose_view).apply {
            setContent {
                MaterialTheme {
                    MyComposeContent()
                }
            }
        }
    }
}
```

---

## AndroidViewBinding

```kotlin
@Composable
fun LegacyLayout() {
    AndroidViewBinding(ItemRowBinding::inflate) {
        titleText.text = "タイトル"
        subtitleText.text = "サブタイトル"
        iconImage.setImageResource(R.drawable.ic_star)
    }
}
```

---

## 状態の共有

```kotlin
@Composable
fun InteropWithState() {
    var text by remember { mutableStateOf("初期値") }

    Column {
        // Compose TextField
        TextField(
            value = text,
            onValueChange = { text = it },
            label = { Text("Compose入力") }
        )

        // AndroidView に状態を反映
        AndroidView(
            factory = { context ->
                TextView(context).apply {
                    textSize = 18f
                }
            },
            update = { textView ->
                textView.text = "View表示: $text"
            }
        )
    }
}
```

---

## まとめ

- `AndroidView`: Compose内にViewを配置（factory + update）
- `ComposeView`: View/Fragment内にComposeを配置
- `AndroidViewBinding`: ViewBinding統合
- `setViewCompositionStrategy`でライフサイクル管理
- 状態共有: Composeの`update`ブロックでViewに反映
- 段階移行: レガシーView → 部分的にCompose導入 → 全面移行

---

8種類のAndroidアプリテンプレート（Compose設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Modifier完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-modifier-guide-2026)
- [WebView実装ガイド](https://zenn.dev/myougatheaxo/articles/android-webview-compose-2026)
- [State管理完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-state-management-2026)
