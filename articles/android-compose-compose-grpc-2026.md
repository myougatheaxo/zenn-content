---
title: "Compose gRPC完全ガイド — Protocol Buffers/Stub生成/ストリーミング/認証"
emoji: "⚡"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "grpc"]
published: true
---

## この記事で学べること

**Compose gRPC**（Protocol Buffers定義、コード生成、Unary/ストリーミングRPC、認証）を解説します。

---

## セットアップ

```groovy
plugins {
    id("com.google.protobuf") version "0.9.4"
}

dependencies {
    implementation("io.grpc:grpc-okhttp:1.63.0")
    implementation("io.grpc:grpc-protobuf-lite:1.63.0")
    implementation("io.grpc:grpc-stub:1.63.0")
    implementation("io.grpc:grpc-kotlin-stub:1.4.1")
}
```

```protobuf
// src/main/proto/user_service.proto
syntax = "proto3";
package com.example.api;

service UserService {
    rpc GetUser(GetUserRequest) returns (UserResponse);
    rpc ListUsers(ListUsersRequest) returns (stream UserResponse);
}

message GetUserRequest { string id = 1; }
message UserResponse { string id = 1; string name = 2; string email = 3; }
message ListUsersRequest { int32 page_size = 1; }
```

---

## gRPCクライアント

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object GrpcModule {
    @Provides
    @Singleton
    fun provideChannel(): ManagedChannel =
        ManagedChannelBuilder.forAddress("api.example.com", 443)
            .useTransportSecurity()
            .build()

    @Provides
    @Singleton
    fun provideUserService(channel: ManagedChannel): UserServiceGrpcKt.UserServiceCoroutineStub =
        UserServiceGrpcKt.UserServiceCoroutineStub(channel)
}

class UserRepository @Inject constructor(
    private val userService: UserServiceGrpcKt.UserServiceCoroutineStub
) {
    suspend fun getUser(id: String): UserResponse =
        userService.getUser(GetUserRequest.newBuilder().setId(id).build())

    fun listUsers(pageSize: Int): Flow<UserResponse> =
        userService.listUsers(ListUsersRequest.newBuilder().setPageSize(pageSize).build())
}
```

---

## Compose連携

```kotlin
@HiltViewModel
class UserViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel() {
    val users = MutableStateFlow<List<UserResponse>>(emptyList())

    init {
        viewModelScope.launch {
            repository.listUsers(20).collect { user ->
                users.update { it + user }
            }
        }
    }
}

@Composable
fun UserListScreen(viewModel: UserViewModel = hiltViewModel()) {
    val users by viewModel.users.collectAsStateWithLifecycle()

    LazyColumn(contentPadding = PaddingValues(16.dp)) {
        items(users) { user ->
            ListItem(
                headlineContent = { Text(user.name) },
                supportingContent = { Text(user.email) }
            )
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| gRPC | 高速RPC通信 |
| Protocol Buffers | スキーマ定義 |
| `CoroutineStub` | Kotlinコルーチン対応 |
| Server Streaming | リアルタイム配信 |

- gRPCはHTTP/2ベースの高速RPC通信
- Protocol Buffersから型安全なStubを自動生成
- `grpc-kotlin-stub`でコルーチン対応のクライアント
- Server Streamingで効率的なリアルタイム配信

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Retrofit](https://zenn.dev/myougatheaxo/articles/android-compose-compose-retrofit-2026)
- [Compose GraphQL](https://zenn.dev/myougatheaxo/articles/android-compose-compose-graphql-2026)
- [Compose Ktor Client](https://zenn.dev/myougatheaxo/articles/android-compose-compose-ktor-client-2026)
