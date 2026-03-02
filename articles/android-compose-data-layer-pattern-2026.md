---
title: "Data Layer設計パターン完全ガイド — Repository/DataSource/DTO/Mapper"
emoji: "🏛️"
type: "tech"
topics: ["android", "kotlin", "architecture", "jetpackcompose"]
published: true
---

## この記事で学べること

**Data Layer設計**（Repository、DataSource、DTO/Entity分離、Mapper、オフラインファースト、テスト）を解説します。

---

## 3層アーキテクチャ

```
UI Layer → Domain Layer → Data Layer
              ↑                ↑
         UseCase          Repository
                          ↓       ↓
                   RemoteDataSource  LocalDataSource
                          ↓              ↓
                       API (Retrofit)   DB (Room)
```

---

## DataSource

```kotlin
// Remote
interface UserRemoteDataSource {
    suspend fun getUsers(): List<UserDto>
    suspend fun getUser(id: String): UserDto
    suspend fun updateUser(id: String, request: UpdateUserRequest): UserDto
}

class UserRemoteDataSourceImpl @Inject constructor(
    private val api: UserApi
) : UserRemoteDataSource {
    override suspend fun getUsers() = api.getUsers()
    override suspend fun getUser(id: String) = api.getUser(id)
    override suspend fun updateUser(id: String, request: UpdateUserRequest) = api.updateUser(id, request)
}

// Local
interface UserLocalDataSource {
    fun getUsers(): Flow<List<UserEntity>>
    suspend fun insertUsers(users: List<UserEntity>)
    suspend fun getUser(id: String): UserEntity?
}

class UserLocalDataSourceImpl @Inject constructor(
    private val dao: UserDao
) : UserLocalDataSource {
    override fun getUsers() = dao.getAll()
    override suspend fun insertUsers(users: List<UserEntity>) = dao.insertAll(users)
    override suspend fun getUser(id: String) = dao.getById(id)
}
```

---

## DTO/Entity/Domain Model

```kotlin
// API レスポンス
@Serializable
data class UserDto(
    val id: String,
    val name: String,
    val email: String,
    @SerialName("avatar_url") val avatarUrl: String?
)

// Room エンティティ
@Entity(tableName = "users")
data class UserEntity(
    @PrimaryKey val id: String,
    val name: String,
    val email: String,
    val avatarUrl: String?,
    val updatedAt: Long = System.currentTimeMillis()
)

// Domain モデル（UIで使う）
data class User(
    val id: String,
    val name: String,
    val email: String,
    val avatarUrl: String?
)
```

---

## Mapper

```kotlin
fun UserDto.toEntity() = UserEntity(
    id = id, name = name, email = email, avatarUrl = avatarUrl
)

fun UserEntity.toDomain() = User(
    id = id, name = name, email = email, avatarUrl = avatarUrl
)

fun UserDto.toDomain() = User(
    id = id, name = name, email = email, avatarUrl = avatarUrl
)
```

---

## Repository（オフラインファースト）

```kotlin
class UserRepositoryImpl @Inject constructor(
    private val remote: UserRemoteDataSource,
    private val local: UserLocalDataSource
) : UserRepository {

    override fun getUsers(): Flow<List<User>> {
        return local.getUsers()
            .map { entities -> entities.map { it.toDomain() } }
            .onStart { refreshUsers() }
    }

    private suspend fun refreshUsers() {
        try {
            val dtos = remote.getUsers()
            local.insertUsers(dtos.map { it.toEntity() })
        } catch (e: IOException) {
            // オフライン時はローカルキャッシュを使用
        }
    }

    override suspend fun getUser(id: String): User {
        return local.getUser(id)?.toDomain()
            ?: remote.getUser(id).also { local.insertUsers(listOf(it.toEntity())) }.toDomain()
    }
}
```

---

## テスト

```kotlin
class UserRepositoryTest {
    private val fakeRemote = FakeUserRemoteDataSource()
    private val fakeLocal = FakeUserLocalDataSource()
    private val repository = UserRepositoryImpl(fakeRemote, fakeLocal)

    @Test
    fun `getUsers returns local data and refreshes from remote`() = runTest {
        fakeRemote.users = listOf(testUserDto)

        repository.getUsers().test {
            val users = awaitItem()
            assertEquals(1, users.size)
            assertEquals("Test User", users[0].name)
        }
    }

    @Test
    fun `getUsers uses cache when offline`() = runTest {
        fakeLocal.insertUsers(listOf(testUserEntity))
        fakeRemote.shouldThrow = true

        repository.getUsers().test {
            val users = awaitItem()
            assertEquals(1, users.size) // ローカルキャッシュから取得
        }
    }
}
```

---

## まとめ

| 要素 | 責務 |
|------|------|
| DataSource | データ取得/保存 |
| DTO/Entity | 層ごとのデータモデル |
| Mapper | モデル間変換 |
| Repository | データソース統合 |

- DataSourceでAPI/DBアクセスを分離
- DTO→Entity→Domain Modelの変換で層の独立性
- オフラインファースト: ローカル優先→バックグラウンド同期
- Fakeでテスタブルなデータ層

---

8種類のAndroidアプリテンプレート（Data Layer設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [クリーンアーキテクチャ](https://zenn.dev/myougatheaxo/articles/android-compose-clean-architecture-2026)
- [Retrofit/OkHttp](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-okhttp-2026)
- [Room CRUD](https://zenn.dev/myougatheaxo/articles/android-compose-room-flow-crud-2026)
