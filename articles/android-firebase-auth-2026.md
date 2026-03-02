---
title: "Firebase Authentication + Compose — メール・Google・匿名ログインの実装"
emoji: "🔐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "firebase"]
published: true
---

## この記事で学べること

**Firebase Authentication**をComposeアプリに統合し、メール認証・Googleログイン・匿名ログインを実装する方法を解説します。

---

## セットアップ

```kotlin
// build.gradle.kts
plugins {
    id("com.google.gms.google-services")
}

dependencies {
    implementation(platform("com.google.firebase:firebase-bom:33.5.0"))
    implementation("com.google.firebase:firebase-auth-ktx")
    // Googleログイン用
    implementation("androidx.credentials:credentials:1.3.0")
    implementation("androidx.credentials:credentials-play-services-auth:1.3.0")
    implementation("com.google.android.libraries.identity.googleid:googleid:1.1.1")
}
```

---

## AuthRepository

```kotlin
class AuthRepository {
    private val auth = Firebase.auth

    val currentUser: FirebaseUser? get() = auth.currentUser

    val authState: Flow<FirebaseUser?> = callbackFlow {
        val listener = FirebaseAuth.AuthStateListener { auth ->
            trySend(auth.currentUser)
        }
        auth.addAuthStateListener(listener)
        awaitClose { auth.removeAuthStateListener(listener) }
    }

    suspend fun signInWithEmail(email: String, password: String): Result<FirebaseUser> {
        return try {
            val result = auth.signInWithEmailAndPassword(email, password).await()
            Result.success(result.user!!)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }

    suspend fun signUpWithEmail(email: String, password: String): Result<FirebaseUser> {
        return try {
            val result = auth.createUserWithEmailAndPassword(email, password).await()
            Result.success(result.user!!)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }

    suspend fun signInAnonymously(): Result<FirebaseUser> {
        return try {
            val result = auth.signInAnonymously().await()
            Result.success(result.user!!)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }

    fun signOut() {
        auth.signOut()
    }
}
```

---

## AuthViewModel

```kotlin
class AuthViewModel(private val repository: AuthRepository) : ViewModel() {
    val authState: StateFlow<FirebaseUser?> = repository.authState
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), repository.currentUser)

    private val _uiState = MutableStateFlow<AuthUiState>(AuthUiState.Idle)
    val uiState: StateFlow<AuthUiState> = _uiState.asStateFlow()

    fun signIn(email: String, password: String) {
        viewModelScope.launch {
            _uiState.value = AuthUiState.Loading
            repository.signInWithEmail(email, password)
                .onSuccess { _uiState.value = AuthUiState.Success }
                .onFailure { _uiState.value = AuthUiState.Error(it.message ?: "エラー") }
        }
    }

    fun signUp(email: String, password: String) {
        viewModelScope.launch {
            _uiState.value = AuthUiState.Loading
            repository.signUpWithEmail(email, password)
                .onSuccess { _uiState.value = AuthUiState.Success }
                .onFailure { _uiState.value = AuthUiState.Error(it.message ?: "エラー") }
        }
    }

    fun signOut() {
        repository.signOut()
        _uiState.value = AuthUiState.Idle
    }
}

sealed class AuthUiState {
    data object Idle : AuthUiState()
    data object Loading : AuthUiState()
    data object Success : AuthUiState()
    data class Error(val message: String) : AuthUiState()
}
```

---

## ログイン画面

```kotlin
@Composable
fun LoginScreen(viewModel: AuthViewModel, onSuccess: () -> Unit) {
    var email by rememberSaveable { mutableStateOf("") }
    var password by rememberSaveable { mutableStateOf("") }
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    LaunchedEffect(uiState) {
        if (uiState is AuthUiState.Success) onSuccess()
    }

    Column(
        Modifier
            .fillMaxSize()
            .padding(24.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Text("ログイン", style = MaterialTheme.typography.headlineMedium)
        Spacer(Modifier.height(32.dp))

        OutlinedTextField(
            value = email,
            onValueChange = { email = it },
            label = { Text("メールアドレス") },
            keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Email),
            modifier = Modifier.fillMaxWidth()
        )
        Spacer(Modifier.height(8.dp))

        OutlinedTextField(
            value = password,
            onValueChange = { password = it },
            label = { Text("パスワード") },
            visualTransformation = PasswordVisualTransformation(),
            modifier = Modifier.fillMaxWidth()
        )
        Spacer(Modifier.height(16.dp))

        Button(
            onClick = { viewModel.signIn(email, password) },
            modifier = Modifier.fillMaxWidth(),
            enabled = uiState !is AuthUiState.Loading
        ) {
            if (uiState is AuthUiState.Loading) {
                CircularProgressIndicator(Modifier.size(20.dp), strokeWidth = 2.dp)
            } else {
                Text("ログイン")
            }
        }

        if (uiState is AuthUiState.Error) {
            Spacer(Modifier.height(8.dp))
            Text(
                (uiState as AuthUiState.Error).message,
                color = MaterialTheme.colorScheme.error
            )
        }
    }
}
```

---

## 認証状態による画面切り替え

```kotlin
@Composable
fun AuthNavigation(viewModel: AuthViewModel) {
    val user by viewModel.authState.collectAsStateWithLifecycle()

    if (user != null) {
        HomeScreen(
            userName = user?.email ?: "匿名",
            onSignOut = { viewModel.signOut() }
        )
    } else {
        LoginScreen(
            viewModel = viewModel,
            onSuccess = { /* authStateが自動更新 */ }
        )
    }
}
```

---

## まとめ

- `Firebase.auth` でメール/Google/匿名認証
- `AuthStateListener` → `callbackFlow` でリアルタイム状態監視
- `sealed class AuthUiState` で画面状態管理
- `collectAsStateWithLifecycle` で安全な状態収集
- Credential Manager API でGoogleログイン（Android 14+推奨）

---

8種類のAndroidアプリテンプレート（認証機能追加可能な設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Hilt DI完全ガイド](https://zenn.dev/myougatheaxo/articles/android-hilt-dependency-injection-2026)
- [Navigation完全ガイド](https://zenn.dev/myougatheaxo/articles/android-navigation-compose-2026)
- [セキュリティチェックリスト](https://zenn.dev/myougatheaxo/articles/android-security-checklist-2026)
