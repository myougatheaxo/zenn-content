---
title: "Kotlin Arrow完全ガイド — Either/Raise/Option/関数型エラーハンドリング"
emoji: "🏹"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "arrow"]
published: true
---

## この記事で学べること

**Kotlin Arrow**（Either、Raise DSL、Option、関数型プログラミング、エラーハンドリング）を解説します。

---

## セットアップ

```groovy
dependencies {
    implementation("io.arrow-kt:arrow-core:1.2.4")
}
```

---

## Either

```kotlin
sealed class AppError {
    data class Network(val cause: Throwable) : AppError()
    data class Validation(val field: String, val message: String) : AppError()
    data object Unauthorized : AppError()
}

class UserRepository @Inject constructor(private val api: UserApi) {
    suspend fun getUser(id: String): Either<AppError, User> = either {
        ensure(id.isNotBlank()) { AppError.Validation("id", "IDは必須です") }
        catch({ api.fetchUser(id) }) { e ->
            raise(AppError.Network(e))
        }
    }

    suspend fun login(email: String, password: String): Either<AppError, AuthToken> = either {
        ensure(email.contains("@")) { AppError.Validation("email", "無効なメール") }
        ensure(password.length >= 8) { AppError.Validation("password", "8文字以上必要") }
        catch({ api.login(email, password) }) { e ->
            raise(AppError.Unauthorized)
        }
    }
}
```

---

## Raise DSL

```kotlin
context(Raise<AppError>)
suspend fun processOrder(orderId: String): OrderResult {
    val order = getOrder(orderId).bind()  // Either → Raise
    ensure(order.items.isNotEmpty()) { AppError.Validation("items", "空の注文") }

    val payment = processPayment(order).bind()
    val shipping = createShipment(order).bind()

    return OrderResult(order, payment, shipping)
}

// ViewModel
@HiltViewModel
class OrderViewModel @Inject constructor(
    private val repository: OrderRepository
) : ViewModel() {
    var state by mutableStateOf<OrderUiState>(OrderUiState.Idle)
        private set

    fun placeOrder(orderId: String) {
        viewModelScope.launch {
            state = OrderUiState.Loading
            either { processOrder(orderId) }.fold(
                ifLeft = { error ->
                    state = when (error) {
                        is AppError.Network -> OrderUiState.Error("通信エラー")
                        is AppError.Validation -> OrderUiState.Error(error.message)
                        AppError.Unauthorized -> OrderUiState.Error("認証エラー")
                    }
                },
                ifRight = { result -> state = OrderUiState.Success(result) }
            )
        }
    }
}
```

---

## Compose連携

```kotlin
@Composable
fun LoginScreen(viewModel: LoginViewModel = hiltViewModel()) {
    var email by remember { mutableStateOf("") }
    var password by remember { mutableStateOf("") }
    val state by viewModel.state.collectAsStateWithLifecycle()

    Column(Modifier.fillMaxSize().padding(16.dp), verticalArrangement = Arrangement.Center) {
        OutlinedTextField(
            value = email, onValueChange = { email = it },
            label = { Text("メール") },
            isError = state is LoginState.ValidationError &&
                (state as LoginState.ValidationError).field == "email",
            modifier = Modifier.fillMaxWidth()
        )
        Spacer(Modifier.height(8.dp))
        OutlinedTextField(
            value = password, onValueChange = { password = it },
            label = { Text("パスワード") },
            visualTransformation = PasswordVisualTransformation(),
            modifier = Modifier.fillMaxWidth()
        )
        Spacer(Modifier.height(16.dp))

        if (state is LoginState.Error) {
            Text((state as LoginState.Error).message, color = MaterialTheme.colorScheme.error)
            Spacer(Modifier.height(8.dp))
        }

        Button(
            onClick = { viewModel.login(email, password) },
            modifier = Modifier.fillMaxWidth(),
            enabled = state !is LoginState.Loading
        ) { Text("ログイン") }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `Either` | 成功/失敗の型安全表現 |
| `Raise` | 型安全エラーDSL |
| `ensure` | 条件チェック |
| `bind` | Either→Raise変換 |

- `Either<Error, Value>`で例外なしのエラーハンドリング
- `Raise` DSLで命令的なスタイルのエラー処理
- `ensure`で事前条件をチェック
- `fold`でUI状態に変換

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Kotlin Result](https://zenn.dev/myougatheaxo/articles/android-compose-kotlin-result-2026)
- [Kotlin SealedClass](https://zenn.dev/myougatheaxo/articles/android-compose-kotlin-sealed-class-2026)
- [Compose CleanArchitecture](https://zenn.dev/myougatheaxo/articles/android-compose-compose-clean-architecture-2026)
