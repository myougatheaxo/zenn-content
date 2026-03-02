---
title: "Hiltマルチモジュール完全ガイド — feature module/DI境界/API公開"
emoji: "🏗️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "hilt"]
published: true
---

## この記事で学べること

**Hiltマルチモジュール**（feature module構成、DI境界設計、API公開パターン）を解説します。

---

## モジュール構成

```
app/          ← @HiltAndroidApp
core/
  ├── data/   ← Repository実装、DB、API
  ├── domain/ ← UseCase、Repository interface
  └── ui/     ← 共通UI（テーマ、コンポーネント）
feature/
  ├── home/   ← HomeScreen、HomeViewModel
  └── detail/ ← DetailScreen、DetailViewModel
```

---

## core:domain（インターフェース定義）

```kotlin
// core/domain
interface UserRepository {
    fun getUsers(): Flow<List<User>>
    suspend fun getUser(id: Long): User
}

class GetUsersUseCase @Inject constructor(
    private val repository: UserRepository
) {
    operator fun invoke(): Flow<List<User>> = repository.getUsers()
}
```

---

## core:data（実装）

```kotlin
// core/data
@Module
@InstallIn(SingletonComponent::class)
abstract class DataModule {
    @Binds
    @Singleton
    abstract fun bindUserRepository(impl: UserRepositoryImpl): UserRepository
}

class UserRepositoryImpl @Inject constructor(
    private val dao: UserDao,
    private val api: UserApi
) : UserRepository {
    override fun getUsers() = dao.getAll()
    override suspend fun getUser(id: Long) = dao.getById(id)
}
```

---

## feature module

```kotlin
// feature/home
@HiltViewModel
class HomeViewModel @Inject constructor(
    private val getUsersUseCase: GetUsersUseCase
) : ViewModel() {
    val users = getUsersUseCase()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())
}

@Composable
fun HomeScreen(
    viewModel: HomeViewModel = hiltViewModel(),
    onUserClick: (Long) -> Unit
) {
    val users by viewModel.users.collectAsStateWithLifecycle()
    LazyColumn {
        items(users) { user ->
            ListItem(
                headlineContent = { Text(user.name) },
                modifier = Modifier.clickable { onUserClick(user.id) }
            )
        }
    }
}
```

---

## まとめ

| レイヤー | 責務 |
|---------|------|
| `core:domain` | インターフェース/UseCase |
| `core:data` | Repository実装/DI Module |
| `feature:*` | ViewModel/Screen |
| `app` | Navigation/HiltAndroidApp |

- `core:domain`にインターフェースを定義
- `core:data`で`@Binds`によるDI実装
- feature moduleは`core:domain`のみに依存
- `@InstallIn(SingletonComponent::class)`でスコープ管理

---

8種類のAndroidアプリテンプレート（マルチモジュール設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Hilt/DI](https://zenn.dev/myougatheaxo/articles/android-compose-hilt-2026)
- [Hiltテスト](https://zenn.dev/myougatheaxo/articles/android-compose-hilt-testing-2026)
- [Koin + Compose](https://zenn.dev/myougatheaxo/articles/android-compose-koin-compose-2026)
