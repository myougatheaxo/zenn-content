---
title: "View→Compose移行ガイド — 段階的移行戦略"
emoji: "🔄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "migration"]
published: true
---

## この記事で学べること

既存のView（XML）ベースアプリからComposeへの**段階的移行戦略**を解説します。

---

## 移行戦略

```
Phase 1: 新規画面をComposeで作成
  → 既存コードに影響なし

Phase 2: 小さいView→Compose置き換え
  → ボタン、カード等のコンポーネント単位

Phase 3: Fragment内にComposeView追加
  → 既存FragmentのUI一部をCompose化

Phase 4: Fragment→Composeスクリーン移行
  → Navigation Compose導入

Phase 5: Activity→SingleActivity+Compose
  → 全画面Compose化完了
```

---

## Phase 1: ComposeViewで混在

```kotlin
// 既存FragmentにComposeを追加
class UserProfileFragment : Fragment() {

    private val viewModel: UserViewModel by viewModels()

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        return LinearLayout(requireContext()).apply {
            orientation = LinearLayout.VERTICAL

            // 既存のXML部分
            addView(inflater.inflate(R.layout.fragment_user_header, null))

            // 新しいCompose部分
            addView(ComposeView(requireContext()).apply {
                setViewCompositionStrategy(
                    ViewCompositionStrategy.DisposeOnViewTreeLifecycleDestroyed
                )
                setContent {
                    MaterialTheme {
                        UserStatsSection(viewModel)
                    }
                }
            })
        }
    }
}

@Composable
fun UserStatsSection(viewModel: UserViewModel) {
    val stats by viewModel.stats.collectAsStateWithLifecycle()

    Row(
        Modifier.fillMaxWidth().padding(16.dp),
        horizontalArrangement = Arrangement.SpaceEvenly
    ) {
        StatItem("投稿", stats.postCount)
        StatItem("フォロワー", stats.followerCount)
        StatItem("フォロー", stats.followingCount)
    }
}
```

---

## Phase 2: ViewModel共有

```kotlin
// 既存のViewModel（変更不要）
class SharedViewModel : ViewModel() {
    private val _items = MutableStateFlow<List<Item>>(emptyList())
    val items: StateFlow<List<Item>> = _items.asStateFlow()

    fun loadItems() {
        viewModelScope.launch {
            _items.value = repository.getItems()
        }
    }
}

// XML側（既存）
class ItemListFragment : Fragment() {
    private val viewModel: SharedViewModel by activityViewModels()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        lifecycleScope.launch {
            viewModel.items.collect { items ->
                adapter.submitList(items)
            }
        }
    }
}

// Compose側（新規）
@Composable
fun ItemDetailScreen(viewModel: SharedViewModel = hiltViewModel()) {
    val items by viewModel.items.collectAsStateWithLifecycle()
    // Compose UI
}
```

---

## Phase 3: Navigation移行

```kotlin
// Fragment Navigation → Compose Navigation

// 段階的: Fragment + Compose画面の混在
NavHost(navController, startDestination = "home") {
    // Compose画面
    composable("home") { HomeScreen() }
    composable("settings") { SettingsScreen() }

    // まだ移行してないFragment
    fragment<UserProfileFragment>("profile") {
        label = "Profile"
    }
}
```

---

## テーマ移行

```kotlin
// XMLテーマ → Compose MaterialTheme

// MdcTheme（Material Design Components テーマブリッジ）
// implementation("com.google.accompanist:accompanist-themeadapter-material3:0.34.0")

@Composable
fun AppWithBridgedTheme(content: @Composable () -> Unit) {
    Mdc3Theme { // XMLテーマを自動変換
        content()
    }
}

// または完全移行
@Composable
fun AppTheme(content: @Composable () -> Unit) {
    MaterialTheme(
        colorScheme = lightColorScheme(
            primary = Color(0xFF6200EE),
            secondary = Color(0xFF03DAC6)
        ),
        typography = Typography(),
        content = content
    )
}
```

---

## RecyclerView → LazyColumn

```kotlin
// Before: RecyclerView + Adapter
class ItemAdapter : ListAdapter<Item, ItemViewHolder>(DiffCallback) {
    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ItemViewHolder {
        val binding = ItemLayoutBinding.inflate(LayoutInflater.from(parent.context), parent, false)
        return ItemViewHolder(binding)
    }
    override fun onBindViewHolder(holder: ItemViewHolder, position: Int) {
        holder.bind(getItem(position))
    }
}

// After: LazyColumn
@Composable
fun ItemList(items: List<Item>, onItemClick: (Item) -> Unit) {
    LazyColumn {
        items(items, key = { it.id }) { item ->
            ItemCard(
                item = item,
                onClick = { onItemClick(item) },
                modifier = Modifier.animateItem()
            )
        }
    }
}
```

---

## 移行チェックリスト

```
□ ComposeView使用時にViewCompositionStrategy設定
□ Fragment内ComposeはDisposeOnViewTreeLifecycleDestroyed
□ StateFlowをcollectAsStateWithLifecycleで収集
□ テーマブリッジ（MdcTheme）で色/フォント統一
□ 既存のViewModel変更なしで再利用
□ Navigation: fragment<>()でFragment画面を維持
□ テスト: 新規Compose画面にComposeTestRule追加
□ パフォーマンス: LazyColumnにkey指定
```

---

## まとめ

- 段階的移行が最も安全（Big Bangリライトは避ける）
- `ComposeView`で既存FragmentにComposeを埋め込み
- `AndroidView`でComposeから既存Viewを利用
- ViewModelはそのまま共有可能
- テーマは`MdcTheme`でブリッジ
- Navigationは`fragment<>()`でFragment混在OK

---

8種類のAndroidアプリテンプレート（Full Compose設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose⇔View相互運用](https://zenn.dev/myougatheaxo/articles/android-compose-compose-interop-2026)
- [Navigation](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-nested-2026)
- [Material3テーマ](https://zenn.dev/myougatheaxo/articles/android-compose-material3-theming-2026)
