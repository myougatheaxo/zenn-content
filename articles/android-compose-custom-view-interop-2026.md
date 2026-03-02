---
title: "View/Compose相互運用完全ガイド — AndroidView/ComposeView/Fragment統合"
emoji: "🔄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "interop"]
published: true
---

## この記事で学べること

**View/Compose相互運用**（AndroidView、ComposeView、Fragment内Compose、既存View再利用）を解説します。

---

## AndroidView（View → Compose）

```kotlin
@Composable
fun MapViewComposable(modifier: Modifier = Modifier) {
    val context = LocalContext.current
    val lifecycleOwner = LocalLifecycleOwner.current

    AndroidView(
        factory = { ctx ->
            MapView(ctx).apply {
                onCreate(null)
                getMapAsync { googleMap ->
                    googleMap.moveCamera(
                        CameraUpdateFactory.newLatLngZoom(LatLng(35.6762, 139.6503), 12f)
                    )
                }
            }
        },
        update = { mapView ->
            // 再コンポーズ時の更新
        },
        modifier = modifier
    )

    DisposableEffect(lifecycleOwner) {
        onDispose { /* cleanup */ }
    }
}

// AdMobバナー広告
@Composable
fun AdBanner(adUnitId: String, modifier: Modifier = Modifier) {
    AndroidView(
        factory = { ctx ->
            AdView(ctx).apply {
                setAdSize(AdSize.BANNER)
                this.adUnitId = adUnitId
                loadAd(AdRequest.Builder().build())
            }
        },
        modifier = modifier.fillMaxWidth()
    )
}
```

---

## ComposeView（Compose → View/Fragment）

```kotlin
// Fragment内でCompose使用
class SettingsFragment : Fragment() {
    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        return ComposeView(requireContext()).apply {
            setViewCompositionStrategy(ViewCompositionStrategy.DisposeOnViewTreeLifecycleDestroyed)
            setContent {
                MaterialTheme {
                    SettingsScreen()
                }
            }
        }
    }
}

// RecyclerView内でCompose使用
class ComposeViewHolder(
    private val composeView: ComposeView
) : RecyclerView.ViewHolder(composeView) {
    fun bind(item: Item) {
        composeView.setContent {
            ItemCard(item = item)
        }
    }
}
```

---

## AndroidViewBinding

```kotlin
@Composable
fun LegacyLayoutInCompose() {
    AndroidViewBinding(LegacyLayoutBinding::inflate) {
        titleText.text = "レガシーView"
        descriptionText.text = "XMLレイアウトをComposeに埋め込み"
        actionButton.setOnClickListener { /* action */ }
    }
}
```

---

## まとめ

| 方向 | API |
|------|-----|
| View → Compose | `AndroidView` |
| XML → Compose | `AndroidViewBinding` |
| Compose → View | `ComposeView` |
| Fragment内 | `setContent { }` |

- `AndroidView`で既存Viewをcomposableに変換
- `ComposeView`でFragment/RecyclerViewにCompose埋め込み
- `ViewCompositionStrategy`でライフサイクル制御
- 段階的なCompose移行に必須のテクニック

---

8種類のAndroidアプリテンプレート（フルCompose）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [WebView](https://zenn.dev/myougatheaxo/articles/android-compose-webview-2026)
- [テスト](https://zenn.dev/myougatheaxo/articles/android-compose-compose-testing-2026)
- [Navigation](https://zenn.dev/myougatheaxo/articles/android-compose-type-safe-navigation-2026)
