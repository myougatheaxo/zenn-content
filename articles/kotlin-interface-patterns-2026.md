---
title: "Kotlinインターフェース実践ガイド — default実装/delegation/DI"
emoji: "🔌"
type: "tech"
topics: ["android", "kotlin", "interface", "patterns"]
published: true
---

## この記事で学べること

Kotlinの**インターフェース**（デフォルト実装、プロパティ、委譲、DI活用）を解説します。

---

## デフォルト実装

```kotlin
interface Logger {
    fun log(message: String)

    // デフォルト実装
    fun info(message: String) = log("[INFO] $message")
    fun warn(message: String) = log("[WARN] $message")
    fun error(message: String) = log("[ERROR] $message")
}

class ConsoleLogger : Logger {
    override fun log(message: String) = println(message)
}

class FileLogger(private val file: File) : Logger {
    override fun log(message: String) {
        file.appendText("$message\n")
    }
}
```

---

## プロパティ付きインターフェース

```kotlin
interface Identifiable {
    val id: String
}

interface Timestamped {
    val createdAt: Long
    val updatedAt: Long
    val isNew: Boolean get() = System.currentTimeMillis() - createdAt < 86400000 // 24h
}

data class Article(
    override val id: String,
    val title: String,
    override val createdAt: Long,
    override val updatedAt: Long
) : Identifiable, Timestamped
```

---

## Repository パターン

```kotlin
interface UserRepository {
    suspend fun getAll(): List<User>
    suspend fun getById(id: String): User?
    suspend fun save(user: User)
    suspend fun delete(id: String)
}

class RoomUserRepository(private val dao: UserDao) : UserRepository {
    override suspend fun getAll() = dao.getAll()
    override suspend fun getById(id: String) = dao.getById(id)
    override suspend fun save(user: User) = dao.upsert(user)
    override suspend fun delete(id: String) = dao.deleteById(id)
}

class FakeUserRepository : UserRepository {
    private val users = mutableListOf<User>()
    override suspend fun getAll() = users.toList()
    override suspend fun getById(id: String) = users.find { it.id == id }
    override suspend fun save(user: User) { users.add(user) }
    override suspend fun delete(id: String) { users.removeAll { it.id == id } }
}
```

---

## 関数型インターフェース (SAM)

```kotlin
fun interface Validator<T> {
    fun validate(value: T): Boolean
}

val emailValidator = Validator<String> { it.contains("@") && it.contains(".") }
val passwordValidator = Validator<String> { it.length >= 8 }

fun <T> T.isValid(validator: Validator<T>): Boolean = validator.validate(this)

// 使用
"user@example.com".isValid(emailValidator) // true
"short".isValid(passwordValidator) // false
```

---

## Hilt DIでの活用

```kotlin
interface AnalyticsService {
    fun trackEvent(name: String, params: Map<String, String> = emptyMap())
    fun trackScreen(screenName: String)
}

class FirebaseAnalyticsService(private val analytics: FirebaseAnalytics) : AnalyticsService {
    override fun trackEvent(name: String, params: Map<String, String>) {
        analytics.logEvent(name, bundleOf(*params.entries.map { it.key to it.value }.toTypedArray()))
    }
    override fun trackScreen(screenName: String) {
        analytics.logEvent(FirebaseAnalytics.Event.SCREEN_VIEW, bundleOf("screen_name" to screenName))
    }
}

@Module
@InstallIn(SingletonComponent::class)
abstract class AnalyticsModule {
    @Binds
    abstract fun bindAnalyticsService(impl: FirebaseAnalyticsService): AnalyticsService
}
```

---

## まとめ

- デフォルト実装で共通ロジックをインターフェースに定義
- プロパティ付きインターフェースで共通契約
- Repositoryパターンで実装を差し替え可能に
- `fun interface`でSAM変換（ラムダでインスタンス化）
- Hilt `@Binds`でインターフェース⇔実装のDI
- テスト時はFake実装に差し替え

---

8種類のAndroidアプリテンプレート（インターフェース設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [sealed classガイド](https://zenn.dev/myougatheaxo/articles/kotlin-sealed-class-2026)
- [Hilt依存性注入ガイド](https://zenn.dev/myougatheaxo/articles/android-hilt-dependency-injection-2026)
- [delegation/委譲ガイド](https://zenn.dev/myougatheaxo/articles/kotlin-delegation-2026)
