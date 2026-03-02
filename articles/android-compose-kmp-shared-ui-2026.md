---
title: "Kotlin Multiplatform + Compose Multiplatform入門ガイド"
emoji: "🌍"
type: "tech"
topics: ["android", "kotlin", "kmp", "multiplatform"]
published: true
---

## この記事で学べること

**Kotlin Multiplatform (KMP)** + Compose MultiplatformでAndroid/iOS/Desktop共通UIを解説します。

---

## プロジェクト構成

```
project/
├── composeApp/
│   ├── src/
│   │   ├── commonMain/    # 共通コード
│   │   ├── androidMain/   # Android固有
│   │   └── iosMain/       # iOS固有
│   └── build.gradle.kts
├── shared/                # ビジネスロジック共有
│   └── src/
│       ├── commonMain/
│       ├── androidMain/
│       └── iosMain/
└── gradle/libs.versions.toml
```

---

## 共通UI (commonMain)

```kotlin
// commonMain/kotlin/App.kt
@Composable
fun App() {
    MaterialTheme {
        var count by remember { mutableIntStateOf(0) }

        Column(
            modifier = Modifier.fillMaxSize(),
            horizontalAlignment = Alignment.CenterHorizontally,
            verticalArrangement = Arrangement.Center
        ) {
            Text("Count: $count", style = MaterialTheme.typography.headlineLarge)
            Spacer(Modifier.height(16.dp))
            Button(onClick = { count++ }) {
                Text("Increment")
            }
        }
    }
}
```

---

## expect/actual パターン

```kotlin
// commonMain: 期待宣言
expect fun getPlatformName(): String

expect class PlatformContext

expect fun createDataStore(context: PlatformContext): DataStore<Preferences>

// androidMain: 実装
actual fun getPlatformName(): String = "Android"

actual typealias PlatformContext = Context

actual fun createDataStore(context: PlatformContext): DataStore<Preferences> {
    return PreferenceDataStoreFactory.create {
        context.filesDir.resolve("settings.preferences_pb")
    }
}

// iosMain: 実装
actual fun getPlatformName(): String = "iOS"
```

---

## 共通ViewModel

```kotlin
// commonMain
class CounterViewModel {
    private val _count = MutableStateFlow(0)
    val count = _count.asStateFlow()

    fun increment() {
        _count.update { it + 1 }
    }
}

// 共通画面で使用
@Composable
fun CounterScreen(viewModel: CounterViewModel = remember { CounterViewModel() }) {
    val count by viewModel.count.collectAsState()

    Column {
        Text("Count: $count")
        Button(onClick = viewModel::increment) {
            Text("+1")
        }
    }
}
```

---

## Ktor Client (共通HTTP)

```kotlin
// commonMain
class ApiClient {
    private val client = HttpClient {
        install(ContentNegotiation) {
            json(Json { ignoreUnknownKeys = true })
        }
    }

    suspend fun getUsers(): List<User> {
        return client.get("https://api.example.com/users").body()
    }
}

// 各プラットフォームのエンジン
// androidMain: implementation("io.ktor:ktor-client-okhttp")
// iosMain: implementation("io.ktor:ktor-client-darwin")
```

---

## まとめ

- KMPでビジネスロジック/UI両方を共通化
- `expect`/`actual`でプラットフォーム固有実装
- Compose Multiplatformで共通UI（Android/iOS/Desktop）
- Ktor Clientで共通HTTP通信
- Kotlinx SerializationでJSON共通処理
- Koinで共通DI（Hiltは非対応）

---

8種類のAndroidアプリテンプレート（Kotlin最新設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Ktor Clientガイド](https://zenn.dev/myougatheaxo/articles/android-compose-ktor-client-2026)
- [Koin DIガイド](https://zenn.dev/myougatheaxo/articles/android-compose-dependency-injection-koin-2026)
- [マルチモジュール](https://zenn.dev/myougatheaxo/articles/android-compose-multi-module-nav-2026)
