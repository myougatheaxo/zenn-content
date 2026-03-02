---
title: "Kotlin Multiplatform + Compose完全ガイド — iOS/Android/Desktop共通UI"
emoji: "🌐"
type: "tech"
topics: ["android", "kotlin", "kmp", "jetpackcompose"]
published: true
---

## この記事で学べること

**Kotlin Multiplatform + Compose Multiplatform**（共通UI、expect/actual、プラットフォーム別実装、Koin DI、ナビゲーション）を解説します。

---

## プロジェクト構成

```
my-kmp-app/
├── composeApp/
│   ├── src/
│   │   ├── commonMain/    # 共通コード + UI
│   │   ├── androidMain/   # Android固有
│   │   ├── iosMain/       # iOS固有
│   │   └── desktopMain/   # Desktop固有
│   └── build.gradle.kts
├── shared/
│   └── src/commonMain/    # ビジネスロジック
└── iosApp/                # iOS Xcode プロジェクト
```

---

## 共通UI

```kotlin
// commonMain
@Composable
fun App() {
    MaterialTheme {
        var count by remember { mutableIntStateOf(0) }

        Column(
            Modifier.fillMaxSize().padding(16.dp),
            horizontalAlignment = Alignment.CenterHorizontally,
            verticalArrangement = Arrangement.Center
        ) {
            Text("Count: $count", style = MaterialTheme.typography.headlineLarge)
            Spacer(Modifier.height(16.dp))
            Button(onClick = { count++ }) {
                Text("Increment")
            }
            Spacer(Modifier.height(8.dp))
            Text("Platform: ${getPlatform().name}")
        }
    }
}
```

---

## expect/actual

```kotlin
// commonMain
expect class Platform() {
    val name: String
}

expect fun getPlatform(): Platform

// androidMain
actual class Platform actual constructor() {
    actual val name: String = "Android ${Build.VERSION.SDK_INT}"
}
actual fun getPlatform(): Platform = Platform()

// iosMain
actual class Platform actual constructor() {
    actual val name: String = UIDevice.currentDevice.systemName() +
        " " + UIDevice.currentDevice.systemVersion
}
actual fun getPlatform(): Platform = Platform()
```

---

## 共通ViewModel

```kotlin
// commonMain
class ItemsViewModel(private val repository: ItemRepository) {
    private val _items = MutableStateFlow<List<Item>>(emptyList())
    val items: StateFlow<List<Item>> = _items

    private val _isLoading = MutableStateFlow(false)
    val isLoading: StateFlow<Boolean> = _isLoading

    fun loadItems() {
        _isLoading.value = true
        CoroutineScope(Dispatchers.Default).launch {
            _items.value = repository.getItems()
            _isLoading.value = false
        }
    }
}

// commonMain - Koin DI
val appModule = module {
    singleOf(::ItemRepository)
    factoryOf(::ItemsViewModel)
}
```

---

## 共通画面

```kotlin
@Composable
fun ItemListScreen(viewModel: ItemsViewModel = koinInject()) {
    val items by viewModel.items.collectAsState()
    val isLoading by viewModel.isLoading.collectAsState()

    LaunchedEffect(Unit) { viewModel.loadItems() }

    if (isLoading) {
        Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
            CircularProgressIndicator()
        }
    } else {
        LazyColumn(Modifier.fillMaxSize(), contentPadding = PaddingValues(16.dp)) {
            items(items, key = { it.id }) { item ->
                Card(Modifier.fillMaxWidth().padding(vertical = 4.dp)) {
                    Text(item.title, Modifier.padding(16.dp))
                }
            }
        }
    }
}
```

---

## プラットフォーム別エントリーポイント

```kotlin
// androidMain
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent { App() }
    }
}

// iosMain
fun MainViewController() = ComposeUIViewController { App() }

// desktopMain
fun main() = application {
    Window(onCloseRequest = ::exitApplication, title = "My KMP App") {
        App()
    }
}
```

---

## まとめ

| 要素 | 場所 |
|------|------|
| 共通UI | `commonMain` |
| ビジネスロジック | `commonMain/shared` |
| プラットフォーム固有 | `androidMain/iosMain` |
| DI | Koin Multiplatform |
| Navigation | Compose Navigation |

- Compose Multiplatformで1つのUIコードからiOS/Android/Desktop対応
- `expect/actual`でプラットフォーム固有の実装を分離
- Koin Multiplatformで共通DI
- 95%以上のコードを共通化可能

---

8種類のAndroidアプリテンプレート（KMP対応可能）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [KMP共通UI](https://zenn.dev/myougatheaxo/articles/android-compose-kmp-shared-ui-2026)
- [Ktor Multiplatform](https://zenn.dev/myougatheaxo/articles/android-compose-ktor-multiplatform-2026)
- [Koin DI](https://zenn.dev/myougatheaxo/articles/android-compose-dependency-injection-koin-2026)
