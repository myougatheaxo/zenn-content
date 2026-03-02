---
title: "Compose MVI完全ガイド — Intent/State/SideEffect/単方向データフロー"
emoji: "🔁"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "architecture"]
published: true
---

## この記事で学べること

**Compose MVI**（Model-View-Intent、単方向データフロー、State管理、SideEffect処理）を解説します。

---

## MVIの構成

```kotlin
// State: UIの状態
data class LoginState(
    val email: String = "",
    val password: String = "",
    val isLoading: Boolean = false,
    val error: String? = null
)

// Intent: ユーザーの意図
sealed class LoginIntent {
    data class EmailChanged(val email: String) : LoginIntent()
    data class PasswordChanged(val password: String) : LoginIntent()
    data object LoginClicked : LoginIntent()
}

// SideEffect: 1回限りのイベント
sealed class LoginEffect {
    data object NavigateToHome : LoginEffect()
    data class ShowToast(val message: String) : LoginEffect()
}
```

---

## ViewModel

```kotlin
@HiltViewModel
class LoginViewModel @Inject constructor(
    private val authRepository: AuthRepository
) : ViewModel() {

    private val _state = MutableStateFlow(LoginState())
    val state: StateFlow<LoginState> = _state.asStateFlow()

    private val _effect = Channel<LoginEffect>()
    val effect: Flow<LoginEffect> = _effect.receiveAsFlow()

    fun onIntent(intent: LoginIntent) {
        when (intent) {
            is LoginIntent.EmailChanged -> {
                _state.update { it.copy(email = intent.email, error = null) }
            }
            is LoginIntent.PasswordChanged -> {
                _state.update { it.copy(password = intent.password, error = null) }
            }
            LoginIntent.LoginClicked -> login()
        }
    }

    private fun login() {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true, error = null) }
            try {
                authRepository.login(_state.value.email, _state.value.password)
                _effect.send(LoginEffect.NavigateToHome)
            } catch (e: Exception) {
                _state.update { it.copy(isLoading = false, error = e.message) }
            }
        }
    }
}
```

---

## Compose UI

```kotlin
@Composable
fun LoginScreen(
    viewModel: LoginViewModel = hiltViewModel(),
    onNavigateToHome: () -> Unit
) {
    val state by viewModel.state.collectAsStateWithLifecycle()
    val context = LocalContext.current

    LaunchedEffect(Unit) {
        viewModel.effect.collect { effect ->
            when (effect) {
                LoginEffect.NavigateToHome -> onNavigateToHome()
                is LoginEffect.ShowToast -> Toast.makeText(context, effect.message, Toast.LENGTH_SHORT).show()
            }
        }
    }

    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(16.dp)) {
        OutlinedTextField(
            value = state.email,
            onValueChange = { viewModel.onIntent(LoginIntent.EmailChanged(it)) },
            label = { Text("メールアドレス") },
            modifier = Modifier.fillMaxWidth(),
            isError = state.error != null
        )
        OutlinedTextField(
            value = state.password,
            onValueChange = { viewModel.onIntent(LoginIntent.PasswordChanged(it)) },
            label = { Text("パスワード") },
            modifier = Modifier.fillMaxWidth(),
            visualTransformation = PasswordVisualTransformation()
        )
        state.error?.let { Text(it, color = MaterialTheme.colorScheme.error) }
        Button(
            onClick = { viewModel.onIntent(LoginIntent.LoginClicked) },
            modifier = Modifier.fillMaxWidth(),
            enabled = !state.isLoading
        ) {
            if (state.isLoading) CircularProgressIndicator(Modifier.size(24.dp))
            else Text("ログイン")
        }
    }
}
```

---

## まとめ

| 概念 | 役割 |
|------|------|
| State | UIの現在状態 |
| Intent | ユーザーアクション |
| SideEffect | 1回限りのイベント |
| `Channel` | SideEffect配信 |

- State → UI → Intent → ViewModel → Stateの単方向フロー
- `StateFlow`でUIの状態を管理
- `Channel`で画面遷移/Toast等の1回限りイベント
- テスタビリティが高い（State入力→State出力）

---

8種類のAndroidアプリテンプレート（MVI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose CleanArchitecture](https://zenn.dev/myougatheaxo/articles/android-compose-compose-clean-architecture-2026)
- [Compose UnidirectionalFlow](https://zenn.dev/myougatheaxo/articles/android-compose-compose-unidirectional-2026)
- [Flow StateFlow](https://zenn.dev/myougatheaxo/articles/android-compose-flow-state-flow-2026)
