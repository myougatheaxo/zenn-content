---
title: "GraphQL + Apollo Client完全ガイド — Query/Mutation/Subscription/Compose連携"
emoji: "🔮"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "graphql"]
published: true
---

## この記事で学べること

**GraphQL + Apollo Client**（Query、Mutation、Subscription、キャッシュ、Compose連携）を解説します。

---

## セットアップ

```kotlin
plugins {
    id("com.apollographql.apollo") version "4.1.0"
}
dependencies {
    implementation("com.apollographql.apollo:apollo-runtime:4.1.0")
}
apollo {
    service("api") {
        packageName.set("com.example.graphql")
        schemaFile.set(file("src/main/graphql/schema.graphqls"))
    }
}
```

---

## スキーマ & クエリ定義

```graphql
# schema.graphqls
type User {
    id: ID!
    name: String!
    email: String!
    posts: [Post!]!
}

type Post {
    id: ID!
    title: String!
    content: String!
    author: User!
}

type Query {
    users: [User!]!
    user(id: ID!): User
    posts(limit: Int, offset: Int): [Post!]!
}

type Mutation {
    createPost(title: String!, content: String!): Post!
    deletePost(id: ID!): Boolean!
}
```

```graphql
# GetUsers.graphql
query GetUsers {
    users {
        id
        name
        email
    }
}

# CreatePost.graphql
mutation CreatePost($title: String!, $content: String!) {
    createPost(title: $title, content: $content) {
        id
        title
    }
}
```

---

## Apollo Client設定

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object ApolloModule {
    @Provides
    @Singleton
    fun provideApolloClient(): ApolloClient {
        return ApolloClient.Builder()
            .serverUrl("https://api.example.com/graphql")
            .addHttpHeader("Authorization", "Bearer token")
            .normalizedCache(MemoryCacheFactory(maxSizeBytes = 10 * 1024 * 1024))
            .build()
    }
}
```

---

## Repository

```kotlin
class UserRepository @Inject constructor(
    private val apolloClient: ApolloClient
) {
    fun getUsers(): Flow<List<User>> = flow {
        val response = apolloClient.query(GetUsersQuery()).execute()
        response.data?.users?.let { users ->
            emit(users.map { User(it.id, it.name, it.email) })
        }
    }

    suspend fun createPost(title: String, content: String): Post? {
        val response = apolloClient.mutation(CreatePostMutation(title, content)).execute()
        return response.data?.createPost?.let { Post(it.id, it.title) }
    }
}
```

---

## Compose画面

```kotlin
@Composable
fun UsersScreen(viewModel: UsersViewModel = hiltViewModel()) {
    val users by viewModel.users.collectAsStateWithLifecycle()
    val isLoading by viewModel.isLoading.collectAsStateWithLifecycle()

    if (isLoading) {
        Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
            CircularProgressIndicator()
        }
    } else {
        LazyColumn(Modifier.fillMaxSize(), contentPadding = PaddingValues(16.dp)) {
            items(users, key = { it.id }) { user ->
                ListItem(
                    headlineContent = { Text(user.name) },
                    supportingContent = { Text(user.email) }
                )
            }
        }
    }
}
```

---

## まとめ

| 操作 | Apollo API |
|------|-----------|
| Query | `apolloClient.query()` |
| Mutation | `apolloClient.mutation()` |
| Subscription | `apolloClient.subscription()` |
| キャッシュ | `normalizedCache()` |

- Apollo Client 4でKotlinファースト設計
- `.graphql`ファイルからコード自動生成
- `normalizedCache`で自動キャッシュ管理
- Kotlinコルーチン/Flow完全対応

---

8種類のAndroidアプリテンプレート（API連携対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Retrofit/OkHttp](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-okhttp-2026)
- [Ktor Client](https://zenn.dev/myougatheaxo/articles/android-compose-ktor-client-2026)
- [オフラインファースト](https://zenn.dev/myougatheaxo/articles/android-compose-offline-first-2026)
