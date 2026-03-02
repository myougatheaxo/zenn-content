---
title: "gRPC Android完全ガイド — Proto定義/Channel管理/ストリーミング"
emoji: "📡"
type: "tech"
topics: ["android", "kotlin", "grpc", "network"]
published: true
---

## この記事で学べること

**gRPC Android**（Proto定義、Channel管理、Unary/ストリーミング、Flow統合、エラーハンドリング）を解説します。

---

## Proto定義

```protobuf
// src/main/proto/user_service.proto
syntax = "proto3";

package com.example.app;

option java_package = "com.example.app.grpc";
option java_multiple_files = true;

service UserService {
    rpc GetUser (GetUserRequest) returns (UserResponse);
    rpc ListUsers (ListUsersRequest) returns (stream UserResponse);
    rpc CreateUser (CreateUserRequest) returns (UserResponse);
}

message GetUserRequest {
    string user_id = 1;
}

message CreateUserRequest {
    string name = 1;
    string email = 2;
}

message UserResponse {
    string id = 1;
    string name = 2;
    string email = 3;
    int64 created_at = 4;
}

message ListUsersRequest {
    int32 page_size = 1;
    string page_token = 2;
}
```

---

## Channel設定

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object GrpcModule {
    @Provides
    @Singleton
    fun provideChannel(): ManagedChannel {
        return ManagedChannelBuilder
            .forAddress("api.example.com", 443)
            .useTransportSecurity()
            .keepAliveTime(30, TimeUnit.SECONDS)
            .keepAliveTimeout(10, TimeUnit.SECONDS)
            .idleTimeout(5, TimeUnit.MINUTES)
            .build()
    }

    @Provides
    @Singleton
    fun provideUserServiceStub(channel: ManagedChannel): UserServiceGrpcKt.UserServiceCoroutineStub {
        return UserServiceGrpcKt.UserServiceCoroutineStub(channel)
    }
}
```

---

## Repository実装

```kotlin
class UserGrpcRepository @Inject constructor(
    private val userStub: UserServiceGrpcKt.UserServiceCoroutineStub
) {
    // Unary RPC
    suspend fun getUser(userId: String): Result<User> {
        return try {
            val request = GetUserRequest.newBuilder()
                .setUserId(userId)
                .build()
            val response = userStub.getUser(request)
            Result.success(response.toDomain())
        } catch (e: StatusException) {
            Result.failure(e.toAppError())
        }
    }

    // Server Streaming → Flow
    fun listUsers(pageSize: Int): Flow<User> = flow {
        val request = ListUsersRequest.newBuilder()
            .setPageSize(pageSize)
            .build()

        userStub.listUsers(request).collect { response ->
            emit(response.toDomain())
        }
    }
}

// エラー変換
fun StatusException.toAppError(): AppError {
    return when (status.code) {
        Status.Code.NOT_FOUND -> AppError.NotFound(status.description ?: "")
        Status.Code.PERMISSION_DENIED -> AppError.Unauthorized
        Status.Code.UNAVAILABLE -> AppError.NetworkError
        else -> AppError.Unknown(status.description ?: "")
    }
}
```

---

## build.gradle設定

```kotlin
plugins {
    id("com.google.protobuf") version "0.9.4"
}

dependencies {
    implementation("io.grpc:grpc-okhttp:1.62.2")
    implementation("io.grpc:grpc-kotlin-stub:1.4.1")
    implementation("io.grpc:grpc-protobuf-lite:1.62.2")
    implementation("com.google.protobuf:protobuf-kotlin-lite:4.26.0")
}

protobuf {
    protoc { artifact = "com.google.protobuf:protoc:4.26.0" }
    plugins {
        create("grpc") { artifact = "io.grpc:protoc-gen-grpc-java:1.62.2" }
        create("grpckt") { artifact = "io.grpc:protoc-gen-grpc-kotlin:1.4.1:jdk8@jar" }
    }
    generateProtoTasks {
        all().forEach { task ->
            task.builtins { create("java") { option("lite") } }
            task.plugins {
                create("grpc") { option("lite") }
                create("grpckt")
            }
        }
    }
}
```

---

## まとめ

| RPC種別 | 用途 |
|---------|------|
| Unary | 単一リクエスト/レスポンス |
| Server Streaming | サーバーから連続データ |
| Client Streaming | クライアントから連続データ |
| Bidirectional | 双方向ストリーム |

- Protocol Buffersで型安全なAPI定義
- Kotlin Coroutine対応のStubで自然な非同期処理
- Server StreamingをFlowに変換してCompose連携
- RESTより高速・型安全なAPI通信

---

8種類のAndroidアプリテンプレート（API通信最適化済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [GraphQL Apollo](https://zenn.dev/myougatheaxo/articles/android-compose-graphql-apollo-2026)
- [Retrofit/OkHttp](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-okhttp-2026)
- [SSE/WebSocket](https://zenn.dev/myougatheaxo/articles/android-compose-sse-websocket-2026)
