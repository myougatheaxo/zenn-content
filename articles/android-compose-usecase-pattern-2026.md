---
title: "UseCase設計パターン完全ガイド — Domain Layer/ビジネスロジック分離"
emoji: "🎯"
type: "tech"
topics: ["android", "kotlin", "architecture", "jetpackcompose"]
published: true
---

## この記事で学べること

**UseCase設計**（単一責任UseCase、Flowベース、エラーハンドリング、テスト、UseCaseの使い分け）を解説します。

---

## UseCase基本形

```kotlin
// 1 UseCase = 1 ビジネスロジック
class GetUserUseCase @Inject constructor(
    private val userRepository: UserRepository
) {
    suspend operator fun invoke(userId: String): User {
        return userRepository.getUser(userId)
    }
}

class GetUsersStreamUseCase @Inject constructor(
    private val userRepository: UserRepository
) {
    operator fun invoke(): Flow<List<User>> {
        return userRepository.getUsers()
    }
}
```

---

## 複数Repositoryを組み合わせ

```kotlin
class GetUserProfileUseCase @Inject constructor(
    private val userRepository: UserRepository,
    private val postRepository: PostRepository,
    private val followRepository: FollowRepository
) {
    suspend operator fun invoke(userId: String): UserProfile {
        return coroutineScope {
            val user = async { userRepository.getUser(userId) }
            val posts = async { postRepository.getUserPosts(userId) }
            val followers = async { followRepository.getFollowerCount(userId) }

            UserProfile(
                user = user.await(),
                posts = posts.await(),
                followerCount = followers.await()
            )
        }
    }
}
```

---

## Result付きUseCase

```kotlin
abstract class ResultUseCase<in P, R> {
    suspend operator fun invoke(params: P): AppResult<R> {
        return try {
            AppResult.Success(execute(params))
        } catch (e: Exception) {
            AppResult.Error(e.toAppError())
        }
    }

    protected abstract suspend fun execute(params: P): R
}

class LoginUseCase @Inject constructor(
    private val authRepository: AuthRepository,
    private val userRepository: UserRepository,
    private val analyticsHelper: AnalyticsHelper
) : ResultUseCase<LoginUseCase.Params, User>() {

    data class Params(val email: String, val password: String)

    override suspend fun execute(params: Params): User {
        val token = authRepository.login(params.email, params.password)
        val user = userRepository.getUser(token.userId)
        analyticsHelper.logEvent("login_success")
        return user
    }
}
```

---

## ViewModelでの使用

```kotlin
@HiltViewModel
class ProfileViewModel @Inject constructor(
    private val getUserProfile: GetUserProfileUseCase,
    savedStateHandle: SavedStateHandle
) : ViewModel() {
    private val userId: String = savedStateHandle.get<String>("userId")!!

    val profile = flow { emit(getUserProfile(userId)) }
        .map<UserProfile, ProfileUiState> { ProfileUiState.Success(it) }
        .catch { emit(ProfileUiState.Error(it.message ?: "Error")) }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), ProfileUiState.Loading)
}
```

---

## UseCaseが不要なケース

```kotlin
// ❌ Repositoryをそのまま呼ぶだけのUseCase → 不要
class GetItemsUseCase(private val repo: ItemRepository) {
    operator fun invoke() = repo.getItems()  // 何もしていない
}

// ✅ ビジネスロジックがある場合のみUseCase化
class GetFilteredItemsUseCase(private val repo: ItemRepository) {
    operator fun invoke(category: String): Flow<List<Item>> {
        return repo.getItems().map { items ->
            items.filter { it.category == category }
                .sortedByDescending { it.priority }
        }
    }
}
```

---

## まとめ

| パターン | 用途 |
|---------|------|
| 単純UseCase | 1Repository操作 |
| 複合UseCase | 複数Repository統合 |
| FlowUseCase | リアクティブデータ |
| ResultUseCase | エラーハンドリング統一 |

- `operator fun invoke`でUseCase(params)形式で呼び出し
- 複数Repositoryの結合ロジックをUseCase化
- Repositoryそのままの薄いラッパーは作らない
- テストではFake Repositoryを注入

---

8種類のAndroidアプリテンプレート（UseCase設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [クリーンアーキテクチャ](https://zenn.dev/myougatheaxo/articles/android-compose-clean-architecture-2026)
- [Data Layer設計](https://zenn.dev/myougatheaxo/articles/android-compose-data-layer-pattern-2026)
- [ViewModel/Hilt](https://zenn.dev/myougatheaxo/articles/android-compose-viewmodel-hilt-2026)
