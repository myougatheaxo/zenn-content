---
title: "DataBinding→Compose移行完全ガイド — XML脱却/段階移行/ComposeView埋め込み"
emoji: "📦"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "migration"]
published: true
---

## この記事で学べること

**DataBinding→Compose移行**（XML脱却、段階移行、ComposeView埋め込み、BindingAdapter代替）を解説します。

---

## 既存DataBinding（移行前）

```xml
<!-- layout/activity_main.xml -->
<layout>
    <data>
        <variable name="viewModel" type="com.example.MainViewModel" />
    </data>

    <LinearLayout android:orientation="vertical">
        <TextView
            android:text="@{viewModel.userName}"
            android:visibility="@{viewModel.isLoggedIn ? View.VISIBLE : View.GONE}" />

        <Button
            android:text="ログイン"
            android:onClick="@{() -> viewModel.login()}" />
    </LinearLayout>
</layout>
```

---

## Compose移行後

```kotlin
@Composable
fun MainScreen(viewModel: MainViewModel = hiltViewModel()) {
    val userName by viewModel.userName.collectAsStateWithLifecycle("")
    val isLoggedIn by viewModel.isLoggedIn.collectAsStateWithLifecycle(false)

    Column(Modifier.padding(16.dp)) {
        if (isLoggedIn) {
            Text(userName)
        }
        Button(onClick = { viewModel.login() }) {
            Text("ログイン")
        }
    }
}
```

---

## 段階的移行（ComposeView埋め込み）

```kotlin
// 既存FragmentにCompose部品を埋め込む
class LegacyFragment : Fragment(R.layout.fragment_legacy) {
    private val viewModel: LegacyViewModel by viewModels()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        // 既存XMLの一部をComposeに置き換え
        view.findViewById<ComposeView>(R.id.compose_container).apply {
            setViewCompositionStrategy(ViewCompositionStrategy.DisposeOnViewTreeLifecycleDestroyed)
            setContent {
                MaterialTheme {
                    // 新しいCompose UI
                    val items by viewModel.items.observeAsState(emptyList())
                    ItemList(items = items)
                }
            }
        }
    }
}

// XMLに<androidx.compose.ui.platform.ComposeView />を追加
```

---

## BindingAdapter代替

```kotlin
// 既存: BindingAdapter
// @BindingAdapter("imageUrl")
// fun loadImage(view: ImageView, url: String?) { ... }

// Compose: 直接使用
@Composable
fun ProfileImage(url: String?, modifier: Modifier = Modifier) {
    AsyncImage(
        model = url,
        contentDescription = null,
        modifier = modifier.clip(CircleShape),
        placeholder = painterResource(R.drawable.placeholder)
    )
}
```

---

## まとめ

| DataBinding | Compose |
|-------------|---------|
| `@{viewModel.field}` | `collectAsStateWithLifecycle()` |
| `android:visibility` | `if (condition)` |
| `android:onClick` | `onClick = { }` |
| `@BindingAdapter` | `@Composable` 関数 |

- DataBindingの双方向バインド → StateFlow + onValueChange
- XML `<include>` → `@Composable` 関数呼び出し
- `ComposeView`で段階的にXMLをCompose化
- BindingAdapterは不要、Composable関数で直接実装

---

8種類のAndroidアプリテンプレート（フルCompose）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [View/Compose相互運用](https://zenn.dev/myougatheaxo/articles/android-compose-custom-view-interop-2026)
- [LiveData→Flow](https://zenn.dev/myougatheaxo/articles/android-compose-live-data-2026)
- [State管理](https://zenn.dev/myougatheaxo/articles/android-compose-compose-state-management-2026)
