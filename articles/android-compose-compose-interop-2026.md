---
title: "Compose⇔View相互運用ガイド — AndroidView/ComposeView"
emoji: "🔀"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "interop"]
published: true
---

## この記事で学べること

Compose⇔Viewの**相互運用**（AndroidView、ComposeView、Fragment統合、段階的移行）を解説します。

---

## AndroidView（ViewをCompose内で使う）

```kotlin
@Composable
fun WebViewInCompose(url: String) {
    AndroidView(
        factory = { context ->
            WebView(context).apply {
                settings.javaScriptEnabled = true
                webViewClient = WebViewClient()
                loadUrl(url)
            }
        },
        update = { webView ->
            webView.loadUrl(url)
        },
        modifier = Modifier.fillMaxSize()
    )
}

// MapView
@Composable
fun MapViewInCompose() {
    val mapView = rememberMapViewWithLifecycle()

    AndroidView(
        factory = { mapView },
        modifier = Modifier.fillMaxSize()
    ) { map ->
        map.getMapAsync { googleMap ->
            googleMap.moveCamera(CameraUpdateFactory.newLatLngZoom(
                LatLng(35.68, 139.76), 12f
            ))
        }
    }
}

// AdView（広告）
@Composable
fun BannerAd(adUnitId: String) {
    AndroidView(
        factory = { context ->
            AdView(context).apply {
                setAdSize(AdSize.BANNER)
                this.adUnitId = adUnitId
                loadAd(AdRequest.Builder().build())
            }
        },
        modifier = Modifier.fillMaxWidth()
    )
}
```

---

## ComposeView（Compose内をView内で使う）

```kotlin
// XMLレイアウト内でCompose
// layout/activity_main.xml
// <androidx.compose.ui.platform.ComposeView
//     android:id="@+id/compose_view"
//     android:layout_width="match_parent"
//     android:layout_height="wrap_content" />

class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        findViewById<ComposeView>(R.id.compose_view).setContent {
            MaterialTheme {
                MyComposableContent()
            }
        }
    }
}

// Fragment内
class MyFragment : Fragment() {
    override fun onCreateView(
        inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?
    ): View {
        return ComposeView(requireContext()).apply {
            setViewCompositionStrategy(
                ViewCompositionStrategy.DisposeOnViewTreeLifecycleDestroyed
            )
            setContent {
                MaterialTheme {
                    FragmentContent()
                }
            }
        }
    }
}
```

---

## RecyclerView内のCompose

```kotlin
class ComposeViewHolder(
    private val composeView: ComposeView
) : RecyclerView.ViewHolder(composeView) {

    fun bind(item: Item) {
        composeView.setContent {
            MaterialTheme {
                ItemCard(item)
            }
        }
    }
}

class MixedAdapter : RecyclerView.Adapter<ComposeViewHolder>() {
    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ComposeViewHolder {
        val composeView = ComposeView(parent.context).apply {
            setViewCompositionStrategy(
                ViewCompositionStrategy.DisposeOnViewTreeLifecycleDestroyed
            )
        }
        return ComposeViewHolder(composeView)
    }

    override fun onBindViewHolder(holder: ComposeViewHolder, position: Int) {
        holder.bind(items[position])
    }
}
```

---

## ViewCompositionStrategy

```kotlin
// Fragment/ViewPager内では必須
composeView.setViewCompositionStrategy(
    // Fragment: Viewが破棄されたらComposition破棄
    ViewCompositionStrategy.DisposeOnViewTreeLifecycleDestroyed
)

// 選択肢:
// DisposeOnDetachedFromWindow（デフォルト）
// DisposeOnDetachedFromWindowOrReleasedFromPool（RecyclerView用）
// DisposeOnLifecycleDestroyed(lifecycle)
// DisposeOnViewTreeLifecycleDestroyed（Fragment推奨）
```

---

## テーマ統合

```kotlin
// View→Composeテーマ変換
@Composable
fun ViewThemeToCompose(content: @Composable () -> Unit) {
    // XMLテーマの色を取得
    val context = LocalContext.current
    val typedValue = TypedValue()
    context.theme.resolveAttribute(android.R.attr.colorPrimary, typedValue, true)
    val primaryColor = Color(typedValue.data)

    MaterialTheme(
        colorScheme = MaterialTheme.colorScheme.copy(primary = primaryColor),
        content = content
    )
}
```

---

## まとめ

| 方向 | API | 用途 |
|------|-----|------|
| View→Compose | `AndroidView` | 既存ViewをCompose内に配置 |
| Compose→View | `ComposeView` | ComposeをXML/Fragment内に配置 |
| テーマ | `MdcTheme` | Material Componentsテーマ変換 |

- `AndroidView`の`factory`で初期化、`update`で更新
- Fragment内の`ComposeView`は`DisposeOnViewTreeLifecycleDestroyed`必須
- RecyclerView内は`DisposeOnDetachedFromWindowOrReleasedFromPool`
- 段階的移行: まずFragment内のComposeView→徐々にCompose化

---

8種類のAndroidアプリテンプレート（Full Compose設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [WebView Bridge](https://zenn.dev/myougatheaxo/articles/android-compose-webview-bridge-2026)
- [カスタムComposable](https://zenn.dev/myougatheaxo/articles/android-compose-custom-composable-2026)
- [Material3テーマ](https://zenn.dev/myougatheaxo/articles/android-compose-material3-theming-2026)
