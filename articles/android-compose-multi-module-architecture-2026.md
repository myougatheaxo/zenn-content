---
title: "マルチモジュール設計ガイド — feature/core/data分割"
emoji: "📦"
type: "tech"
topics: ["android", "kotlin", "architecture", "gradle"]
published: true
---

## この記事で学べること

Androidの**マルチモジュール設計**（feature/core/data分割、依存関係、Convention Plugin）を解説します。

---

## モジュール構成

```
app/
├── app/                    # Application、MainActivity
├── feature/
│   ├── feature-home/       # ホーム画面
│   ├── feature-search/     # 検索画面
│   └── feature-profile/    # プロフィール画面
├── core/
│   ├── core-ui/            # 共通Composable、Theme
│   ├── core-data/          # Repository実装
│   ├── core-domain/        # UseCase、Entity
│   ├── core-network/       # Retrofit、API
│   ├── core-database/      # Room、DAO
│   ├── core-model/         # データモデル（共有）
│   └── core-common/        # ユーティリティ
└── build-logic/            # Convention Plugin
```

---

## 依存関係ルール

```kotlin
// app/build.gradle.kts
dependencies {
    implementation(project(":feature:feature-home"))
    implementation(project(":feature:feature-search"))
    implementation(project(":feature:feature-profile"))
    implementation(project(":core:core-ui"))
}

// feature-home/build.gradle.kts
dependencies {
    implementation(project(":core:core-ui"))
    implementation(project(":core:core-domain"))
    implementation(project(":core:core-model"))
    // ❌ feature同士は依存しない
    // implementation(project(":feature:feature-profile"))
}

// core-data/build.gradle.kts
dependencies {
    implementation(project(":core:core-network"))
    implementation(project(":core:core-database"))
    implementation(project(":core:core-domain"))
    implementation(project(":core:core-model"))
}

// core-domain/build.gradle.kts
dependencies {
    implementation(project(":core:core-model"))
    // ❌ domain は data/network に依存しない
}
```

---

## core-model

```kotlin
// 全モジュールから参照可能なデータモデル
data class User(
    val id: String,
    val name: String,
    val email: String,
    val avatarUrl: String?
)

data class Post(
    val id: String,
    val title: String,
    val content: String,
    val authorId: String,
    val createdAt: Long
)
```

---

## core-domain（UseCase）

```kotlin
class GetUserUseCase @Inject constructor(
    private val userRepository: UserRepository
) {
    operator fun invoke(userId: String): Flow<User> {
        return userRepository.observeUser(userId)
    }
}

class SearchPostsUseCase @Inject constructor(
    private val postRepository: PostRepository
) {
    operator fun invoke(query: String): Flow<List<Post>> {
        return postRepository.searchPosts(query)
            .map { posts -> posts.sortedByDescending { it.createdAt } }
    }
}

// Repositoryインターフェース（domain層に定義）
interface UserRepository {
    fun observeUser(userId: String): Flow<User>
    suspend fun updateUser(user: User)
}
```

---

## feature モジュール

```kotlin
// feature-home/
@HiltViewModel
class HomeViewModel @Inject constructor(
    private val getUserUseCase: GetUserUseCase,
    private val getPostsUseCase: GetPostsUseCase
) : ViewModel() {

    val uiState: StateFlow<HomeUiState> = combine(
        getUserUseCase(currentUserId),
        getPostsUseCase()
    ) { user, posts ->
        HomeUiState.Success(user = user, posts = posts)
    }.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), HomeUiState.Loading)
}

// Navigation（feature内に定義）
fun NavGraphBuilder.homeScreen(onNavigateToDetail: (String) -> Unit) {
    composable("home") {
        HomeScreen(onNavigateToDetail = onNavigateToDetail)
    }
}

// app/でNavGraphをまとめる
NavHost(navController, startDestination = "home") {
    homeScreen(onNavigateToDetail = { navController.navigate("detail/$it") })
    searchScreen()
    profileScreen()
}
```

---

## Convention Plugin

```kotlin
// build-logic/convention/src/main/kotlin/AndroidFeaturePlugin.kt
class AndroidFeaturePlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            pluginManager.apply {
                apply("com.android.library")
                apply("org.jetbrains.kotlin.android")
                apply("com.google.dagger.hilt.android")
                apply("com.google.devtools.ksp")
            }

            dependencies {
                add("implementation", project(":core:core-ui"))
                add("implementation", project(":core:core-domain"))
                add("implementation", project(":core:core-model"))
                add("implementation", libs.findLibrary("hilt.android").get())
                add("ksp", libs.findLibrary("hilt.compiler").get())
                add("implementation", libs.findLibrary("compose.navigation").get())
            }
        }
    }
}

// feature-home/build.gradle.kts
plugins {
    id("myapp.android.feature") // Convention Plugin適用
}
```

---

## まとめ

| レイヤー | 役割 | 依存先 |
|---------|------|--------|
| `app` | DI統合、Navigation | feature、core-ui |
| `feature` | 画面単位の機能 | core-ui、core-domain、core-model |
| `core-domain` | ビジネスロジック | core-model |
| `core-data` | Repository実装 | core-network、core-database、core-domain |
| `core-model` | 共有データモデル | なし |

- feature同士は直接依存しない（app層でNavigation連携）
- domain層はdata/networkに依存しない（依存性逆転）
- Convention Pluginでbuild.gradle.ktsの重複排除
- `operator fun invoke()`でUseCase呼び出しを簡潔に

---

8種類のAndroidアプリテンプレート（マルチモジュール対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Clean Architecture](https://zenn.dev/myougatheaxo/articles/android-compose-clean-architecture-2026)
- [Hilt DI](https://zenn.dev/myougatheaxo/articles/android-compose-dependency-injection-hilt-2026)
- [build.gradle.kts Tips](https://zenn.dev/myougatheaxo/articles/android-compose-build-gradle-tips-2026)
