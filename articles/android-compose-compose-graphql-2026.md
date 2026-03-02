---
title: "Compose GraphQL完全ガイド — Apollo Client/クエリ/ミューテーション/サブスクリプション"
emoji: "◆"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "graphql"]
published: true
---

## この記事で学べること

**Compose GraphQL**（Apollo Client、クエリ、ミューテーション、キャッシュ、Compose連携）を解説します。

---

## セットアップ

```groovy
plugins {
    id("com.apollographql.apollo3") version "3.8.5"
}

dependencies {
    implementation("com.apollographql.apollo3:apollo-runtime:3.8.5")
}

apollo {
    service("api") {
        packageName.set("com.example.app.graphql")
    }
}
```

```graphql
# src/main/graphql/schema.graphqls
type Query {
    users: [User!]!
    user(id: ID!): User
}

type User {
    id: ID!
    name: String!
    email: String!
    posts: [Post!]!
}
```

---

## クエリ

```graphql
# src/main/graphql/GetUsers.graphql
query GetUsers {
    users {
        id
        name
        email
    }
}

query GetUser($id: ID!) {
    user(id: $id) {
        id
        name
        email
        posts {
            id
            title
        }
    }
}
```

```kotlin
class UserRepository @Inject constructor(
    private val apolloClient: ApolloClient
) {
    suspend fun getUsers(): List<GetUsersQuery.User> {
        val response = apolloClient.query(GetUsersQuery()).execute()
        return response.dataOrThrow().users
    }

    suspend fun getUser(id: String): GetUserQuery.User {
        val response = apolloClient.query(GetUserQuery(id)).execute()
        return response.dataOrThrow().user ?: throw Exception("User not found")
    }
}
```

---

## ミューテーション

```graphql
mutation CreateUser($input: CreateUserInput!) {
    createUser(input: $input) {
        id
        name
        email
    }
}
```

```kotlin
suspend fun createUser(name: String, email: String): CreateUserMutation.CreateUser {
    val input = CreateUserInput(name = name, email = email)
    val response = apolloClient.mutation(CreateUserMutation(input)).execute()
    return response.dataOrThrow().createUser
}
```

---

## Compose連携

```kotlin
@HiltViewModel
class UsersViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel() {
    private val _users = MutableStateFlow<List<GetUsersQuery.User>>(emptyList())
    val users = _users.asStateFlow()
    val isLoading = MutableStateFlow(true)

    init {
        viewModelScope.launch {
            isLoading.value = true
            _users.value = repository.getUsers()
            isLoading.value = false
        }
    }
}

@Composable
fun UsersScreen(viewModel: UsersViewModel = hiltViewModel()) {
    val users by viewModel.users.collectAsStateWithLifecycle()
    val isLoading by viewModel.isLoading.collectAsStateWithLifecycle()

    if (isLoading) {
        Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
            CircularProgressIndicator()
        }
    } else {
        LazyColumn(contentPadding = PaddingValues(16.dp)) {
            items(users) { user ->
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

| API | 用途 |
|-----|------|
| `ApolloClient` | GraphQLクライアント |
| `query` | データ取得 |
| `mutation` | データ変更 |
| `subscription` | リアルタイム通知 |

- Apollo ClientでGraphQLスキーマから型安全なコード生成
- `.graphql`ファイルでクエリ/ミューテーションを定義
- `normalizedCache`でキャッシュ管理
- `subscription`でリアルタイムデータを受信

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Retrofit](https://zenn.dev/myougatheaxo/articles/android-compose-compose-retrofit-2026)
- [Compose Ktor Client](https://zenn.dev/myougatheaxo/articles/android-compose-compose-ktor-client-2026)
- [Compose WebSocket](https://zenn.dev/myougatheaxo/articles/android-compose-compose-websocket-2026)
